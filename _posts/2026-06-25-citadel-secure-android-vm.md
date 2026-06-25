---
layout: post
title: "Citadel: A Secure Virtual Machine for Android"
date: 2026-06-25
description: "How we built a cryptographically-protected execution environment for Android — shipping sensitive business logic in an encrypted, tamper-evident, app-identity-bound form using Rust, AES-256-GCM, Argon2id, Ed25519, and white-box cryptography."
---

Android APKs are ZIP archives. Anyone can unpack them, read the bytecode, extract native libraries, and re-sign the result. For most apps this is fine — but for apps that carry sensitive business logic (licence checks, cryptographic keys, proprietary algorithms), it is a fundamental problem.

**Citadel** is a secure virtual machine built to solve that problem. It lets you ship firmware — bytecode that encodes your sensitive logic — inside an APK in a form that is encrypted, tied to the app's identity, and verified at every layer before a single instruction runs.

---

## The Problem

Consider what an attacker with a copy of your APK can do today:

- **Unpack and read** every asset, resource, and DEX class
- **Decompile** Java/Kotlin to near-source via tools like jadx
- **Extract** native `.so` libraries and disassemble them
- **Patch** any of the above, re-sign with their own certificate, and sideload
- **Attach a debugger** (Frida, Xposed) to a running process and inspect memory at will
- **Run on a rooted device or emulator** with full kernel control

Standard Android protections — ProGuard, R8, even basic native code — slow this down but do not stop a determined attacker. Once firmware is on the device, the race is between your obfuscation and their patience.

### What We Need

A solution must satisfy four properties simultaneously:

1. **Confidentiality** — the firmware cannot be read from the APK without secrets that live off-device
2. **Identity binding** — the firmware cannot be transplanted to a different app or re-signed APK
3. **Integrity** — any modification to the assets or the native library is detected before execution
4. **Environment checks** — execution is refused on debugged, rooted, or emulated devices

No single technique achieves all four. Citadel combines several into a layered chain of trust.

---

## High-Level Architecture

Citadel is built around three binary assets that ship inside the APK:

```
┌─────────────────────────────────────────────────────────────────┐
│                        APK Contents                             │
│                                                                 │
│  assets/                                                        │
│  ├── codesign.bin   ← Ed25519 signature over the other two     │
│  ├── license.bin    ← Encrypted identity + secrets             │
│  └── firmware.bin   ← Encrypted bytecode (your logic)          │
│                                                                 │
│  lib/arm64-v8a/                                                 │
│  └── libcitadel.so  ← Rust native library (two integrity slots) │
└─────────────────────────────────────────────────────────────────┘
```

These three assets form a **chain of trust**: each one is verified using secrets obtained from the previous one. You cannot reach the firmware without passing through the signature check and the license decryption first.

### Build-Time vs. Runtime

The overall flow is split between two phases:

```
BUILD TIME (vendor machine)
─────────────────────────────────────────────────────────────────

  Release keystore                 Ed25519 keypair
       │                                │
       ▼                                │
  Extract cert (DER)                    │
       │                                │
       ▼                                ▼
  gen_assets tool ──────────────────────┤
       │                                │
       │  ┌─────────────────────────┐   │
       │  │ FirmwareLicense         │   │
       │  │  package_id             │   │
       │  │  cert_sha256            │   │
       │  │  firmware_sha256        │   │
       │  │  firmware_secret (32B)  │   │
       │  │  customer_secret (32B)  │   │
       │  │  opcode_seed (32B)      │   │
       │  │  valid_until            │   │
       │  └────────────┬────────────┘   │
       │               │                │
       ▼               ▼                ▼
  firmware.bin    license.bin      codesign.bin
  (AES-256-GCM)  (AES-256-GCM)    (Ed25519 sig)
       │               │                │
       └───────────────┴────────────────┘
                       │
                  Place in APK
                       │
               patch_so tool
          (embed SHA-256 + HMAC slots
           into libcitadel.so RX segment)

─────────────────────────────────────────────────────────────────
RUNTIME (device)
─────────────────────────────────────────────────────────────────

  Kotlin: vm.startFromAssets(context)
       │
       ▼
  Read package_id from /proc/self/cmdline
  Read signing cert from APK ZIP META-INF/*.RSA
  Read installer via PackageManager JNI
       │
       ▼
  [1] Verify Ed25519 signature (codesign.bin)
       │
       ▼
  [2] Derive license key: Argon2id(cert + package + vendor_secret)
      Decrypt license.bin → FirmwareLicense
       │
       ▼
  [3] Validate identity constraints
      (package_id, cert_hash, installer, valid_until)
       │
       ▼
  [4] Verify .so SHA-256 slot (early, keyless)
       │
       ▼
  [5] Run 18 environment checks
      (debugger, root, emulator, injected libs)
       │
       ▼
  [6] Derive firmware key: Argon2id(firmware_secret + identity)
      Decrypt firmware.bin → bytecode
       │
       ▼
  [7] Verify firmware hash against license.firmware_sha256
       │
       ▼
  [8] Verify .so HMAC slot (keyed from firmware_secret)
       │
       ▼
  [9] Remap opcodes using license.opcode_seed
      Initialise customer data key (Keystore → WBC → software)
       │
       ▼
  START_OK → Kotlin: vm.run() → result: i64
```

Every step is a hard gate. A failure at any point returns an error code and halts — no partial execution, no fallback to a weaker path.

---

## Design

### The Three-Asset Chain

**codesign.bin** is an Ed25519 signature over the concatenation of the license hash and firmware hash. It is verified first, using a public key baked (and obfuscated) into the `.so`. This gives us a cryptographic guarantee that the assets were produced by the holder of the private key — a secret that never leaves the build server.

**license.bin** carries the secrets needed to decrypt the firmware and a set of identity constraints. It is encrypted with a key derived from:

```
license_key = Argon2id(
    password = sha256(cert) || package_id || LICENSE_EMBED_SECRET,
    salt     = sha256(cert),
    m = 64 MB, t = 3, p = 1
)
```

`LICENSE_EMBED_SECRET` is a 32-byte vendor constant XOR-obfuscated inside the `.so`. This means the license cannot be brute-forced offline without first reversing the native library to extract that constant. The Argon2id parameters (OWASP 2021 minimum) add roughly 2 seconds per attempt even then.

**firmware.bin** is the actual bytecode, encrypted with a key derived from `firmware_secret` (which lives inside the license) and the app's identity string. An attacker who extracts the firmware ciphertext still needs the firmware_secret, which is itself encrypted inside the license.

### Key Derivation Tree

```
                   LICENSE_EMBED_SECRET (in .so)
                            │
              ┌─────────────┴───────────────┐
         sha256(cert)               package_id
              │                         │
              └──────────┬──────────────┘
                         │
                     Argon2id
                         │
                   license_key ──► decrypt license.bin
                                         │
                          ┌──────────────┼──────────────┐
                   firmware_secret  customer_secret  opcode_seed
                          │               │               │
                     Argon2id         Argon2id      Fisher-Yates
                     + identity       + identity     shuffle(25 opcodes)
                          │               │               │
                   firmware_key    customer_key     opcode_table
                          │               │               │
                   decrypt           encrypt/        remap bytecode
                   firmware.bin      decrypt data    at parse time
```

No raw key material appears in any asset file. Every key is derived at runtime, inside native code, from secrets that require a previous decryption step to obtain.

### Identity Binding

The license contains:

```
package_id       = "com.yourcompany.yourapp"
cert_sha256      = sha256(APK signing certificate DER)
installer_policy = "required:com.android.vending"  (optional)
valid_until      = 1893456000                       (Unix timestamp)
```

The license decryption key is computed from the cert hash and package ID. If an attacker extracts the assets and drops them into a different app (different package name, different signing key), the Argon2id derivation produces a different key and decryption fails. There is no flag to flip, no version check to patch — the mismatch is cryptographic.

### Two-Layer Integrity for the Native Library

The `.so` file carries two integrity slots embedded in its RX (executable) segment by the `patch_so` build tool:

```
┌───────────────────────────────────────────────────┐
│ libcitadel.so RX segment                          │
│                                                   │
│  [code] [code] ... [SHA-256 slot 32B] ... [code] │
│                    ↑                              │
│  Checked BEFORE Argon2id work (fast, keyless)     │
│  Catches: casual patchers, byte-flippers          │
│                                                   │
│  [code] [code] ... [HMAC slot 32B]   ... [code]  │
│                    ↑                              │
│  Checked AFTER license decryption                 │
│  Key = firmware_secret (from license)             │
│  Catches: attackers who cleared the SHA-256 slot  │
│  Cannot forge without license decryption first    │
└───────────────────────────────────────────────────┘
```

The early SHA-256 check costs nothing and catches anyone who patched the binary. The late HMAC check is cryptographically bound to the license — even if an attacker knows where the HMAC slot is, they cannot compute a valid value without `firmware_secret`, which requires passing the license decryption step first.

### Per-License Opcode Remapping

The license's `opcode_seed` is a 32-byte random value used to shuffle the 25 opcodes of the instruction set via Fisher-Yates:

```
Canonical opcode table:
  0x00 = Halt    0x01 = PushI64   0x02 = Add   ...

Per-license opcode table (example shuffle):
  0x00 = Mul     0x01 = Halt      0x02 = Sub   ...
```

Firmware is compiled against the shuffled table. The same bytecode in a different customer's license is meaningless — you would need both the seed and the bytecode to understand what the program does. This forces a static analysis attacker to reverse-engineer the shuffle before they can even read the instruction stream.

---

## Implementation

### The Stack Machine (`vm.rs`)

Citadel executes firmware on a simple stack machine with:

- **25 opcodes** — arithmetic, bitwise, stack manipulation, control flow, comparison
- **16 general-purpose i64 registers**
- **Evaluation stack** — max 1,024 values (~8 KiB)
- **Call stack** — max 256 frames
- **Step counter** — max 100,000 instructions per `run()` call
- **Periodic debugger checks** — every 10,000 steps

```
VM Architecture:

  ┌─────────────────────────────────────────────────────┐
  │  Execution Engine                                   │
  │                                                     │
  │   Instruction Pointer ──► fetch opcode              │
  │                               │                     │
  │                          remap via                  │
  │                          opcode_table               │
  │                               │                     │
  │                          dispatch                   │
  │                         /    |    \                 │
  │                      Push   Add   Call  ...         │
  │                        │     │      │               │
  │                     Evaluation   Call Stack         │
  │                       Stack      (max 256 frames)   │
  │                     [v0, v1 ..]  [frame0, frame1..] │
  │                        │                            │
  │                     Registers [r0 .. r15]           │
  │                                                     │
  │   every 10,000 steps ──► is_debugger_attached()?    │
  │   step > 100,000     ──► StepLimitExceeded          │
  └─────────────────────────────────────────────────────┘
```

The execution loop is intentionally minimal. The VM does not do I/O, syscalls, or dynamic memory allocation. Its sole job is to compute a result from sealed firmware, deterministically, in bounded time.

### Environment Checks (`environment.rs`)

Eighteen checks are applied at startup and periodically during execution. They are grouped into three categories:

```
  Debugger Detection (6 checks)
  ├── /proc/self/status  → TracerPid > 0
  ├── /proc/self/stat    → process state 't' or 'T'
  ├── /proc/self/wchan   → contains "ptrace_stop"
  ├── /proc/self/maps    → frida / xposed / substrate /
  │                         magisk / cydia found
  ├── $LD_PRELOAD        → non-empty
  └── .so load path      → unexpected location

  Root Detection (7 checks)
  ├── ro.debuggable      → "1"
  ├── ro.secure          → "0"
  ├── build tags         → "test-keys"
  ├── /system/app/       → Superuser.apk present
  ├── /system/xbin/su    → exists
  ├── /sbin/su           → exists
  └── verified boot      → not "green"

  Emulator Detection (5 checks)
  ├── ro.product.model   → "sdk" / "emulator"
  ├── ro.hardware        → "goldfish" / "ranchu"
  ├── ro.kernel.qemu     → "1"
  ├── /dev/socket/qemud  → exists
  └── CPU speed test     → suspiciously fast
```

All detection strings are obfuscated at compile time via `obfstr::obfstr!` — they are XOR-encrypted in the binary and decrypted on the stack at the moment of use, never appearing in `.rodata`.

### Encryption Pipeline (`firmware.rs`)

Both asset layers use AES-256-GCM with a 96-bit random nonce per file, and keys derived via Argon2id. The layering means each decryption reveals the secrets needed for the next:

```
  codesign.bin
    └─► Ed25519 verify (public key in .so)
            │
            ▼ (passes)
  license.bin
    └─► Argon2id(cert + package + EMBED_SECRET) → license_key
    └─► AES-256-GCM decrypt → FirmwareLicense { firmware_secret,
                                                 customer_secret,
                                                 opcode_seed, ... }
            │
            ▼
  firmware.bin
    └─► Argon2id(firmware_secret + identity) → firmware_key
    └─► AES-256-GCM decrypt → raw bytecode
    └─► SHA-256(bytecode) == license.firmware_sha256 ✓
```

The nonces are stored in the ciphertext files. The Argon2id parameters are intentionally expensive:

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `m_cost`  | 64 MB | OWASP 2021 minimum; kills GPU parallelism |
| `t_cost`  | 3     | ~2 seconds on a single core |
| `p_cost`  | 1     | No parallel benefit on a single device |

### White-Box AES (`wbc.rs`)

On devices without a hardware Keystore (TEE / StrongBox), the customer data key would ordinarily sit in process memory. White-box AES replaces that with Chow-style T-tables — a representation of AES-256 in which the round key material is entangled with the lookup tables themselves:

```
  Standard AES-256:
    plaintext ──► [AddRoundKey][SubBytes][ShiftRows][MixColumns] × 13 ──► ciphertext
                         ↑
                   raw key in memory

  White-Box AES-256 (T-tables):
    plaintext ──► encoded ──► [T0][T1][T2][T3] × 13 ──► decoded ──► ciphertext
                               ↑
                   key baked into 57 KB of tables
                   no raw key appears anywhere in memory
```

Each table entry encodes a combination of the AES operations with a per-key, per-round affine transformation over GF(2). A memory dump reveals the tables but not the key — recovering the key from the tables is a significant cryptanalytic problem (Billet et al.).

The tables can be pre-generated at build time (Codex variant) and compressed inside the license, or generated on first use. At ~57 KB per instance they are compact enough to keep in memory for the app's lifetime.

### Secure Storage (`storage.rs`)

Citadel provides a key-value store for persistent secrets (API tokens, OAuth refresh tokens, etc.) with the following properties:

```
  vm.storeSecret("oauth_token", tokenBytes, passphrase)
         │
         ▼
  key_id   = HMAC-SHA-256(customer_key, "oauth_token")
             (key names never appear in plaintext in the blob)
  salt     = random 16 bytes
  record_key = Argon2id(passphrase + customer_key, salt, m=64 MB)
  ciphertext = AES-256-GCM(record_key, nonce=random 12B, plaintext)
         │
         ▼
  SVMSTORE03: [magic][key_id][salt][nonce][ciphertext]
```

Each record has its own independent salt and nonce, so records are unlinkable without the customer key. The passphrase adds a second factor for particularly sensitive values; the key ID uses an HMAC so that even the key name is not recoverable without the customer key.

---

## Security Layers Summary

```
  ┌─────────────────────────────────────────────────────────┐
  │ Layer 1: Code Signature (Ed25519)                       │
  │   Attacker cannot forge assets without private key      │
  │                                                         │
  │  ┌───────────────────────────────────────────────────┐  │
  │  │ Layer 2: Identity Binding (Argon2id + cert hash)  │  │
  │  │   License decryption fails on different app/cert  │  │
  │  │                                                   │  │
  │  │  ┌─────────────────────────────────────────────┐  │  │
  │  │  │ Layer 3: .so Integrity (SHA-256 + HMAC)     │  │  │
  │  │  │   Detects patched native library             │  │  │
  │  │  │                                             │  │  │
  │  │  │  ┌───────────────────────────────────────┐  │  │  │
  │  │  │  │ Layer 4: Environment Checks (×18)     │  │  │  │
  │  │  │  │   Refuses to run under debugger/root   │  │  │  │
  │  │  │  │                                       │  │  │  │
  │  │  │  │  ┌─────────────────────────────────┐  │  │  │  │
  │  │  │  │  │ Layer 5: Firmware Encryption    │  │  │  │  │
  │  │  │  │  │   Bytecode never in plaintext   │  │  │  │  │
  │  │  │  │  │                                 │  │  │  │  │
  │  │  │  │  │  ┌───────────────────────────┐  │  │  │  │  │
  │  │  │  │  │  │ Layer 6: Opcode Shuffle   │  │  │  │  │  │
  │  │  │  │  │  │   Unique per license      │  │  │  │  │  │
  │  │  │  │  │  └───────────────────────────┘  │  │  │  │  │
  │  │  │  │  └─────────────────────────────────┘  │  │  │  │
  │  │  │  └───────────────────────────────────────┘  │  │  │
  │  │  └─────────────────────────────────────────────┘  │  │
  │  └───────────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────────────┘
```

An attacker must break all six layers to reach executable firmware in a form they can understand. Each layer that fails returns the same generic error code — there is no information leakage about which check failed.

---

## Usage

### Build-Time Workflow

**1. Generate an Ed25519 keypair** (once, store the private key in a secrets manager or HSM):

```bash
cargo run --bin keygen
# → private_key.hex  (store securely)
# → public_key.hex   (embed in .so via keys.rs)
```

**2. Write your firmware** in Rust using the instruction builder:

```rust
use citadel::{Instruction, Program};

let firmware = Program::new(vec![
    Instruction::PushI64(40),
    Instruction::PushI64(2),
    Instruction::Add,
    Instruction::Halt,
])?.to_bytes();
```

**3. Create `licensepack.json`**:

```json
{
  "key":              "<32-byte firmware_secret hex>",
  "value":            "<32-byte customer_secret hex>",
  "cert":             "<release signing cert DER hex>",
  "id":               "com.yourcompany.yourapp",
  "installer_policy": "required:com.android.vending",
  "valid_until":      1893456000
}
```

**4. Run the asset generator**:

```bash
export CODESIGN_PRIVATE_KEY="<private key hex>"
cargo run --manifest-path tools/gen_assets/Cargo.toml
# Writes: license.bin, firmware.bin, codesign.bin
# Prints: FIRMWARE_SECRET=<hex>  (needed for next step)
```

**5. Build and patch the native library**:

```bash
cargo ndk -t arm64-v8a -t armeabi-v7a \
  -o app/src/main/jniLibs \
  build --release \
  --features jni,enforce_patch,enforce_codesign_key

export FIRMWARE_SECRET="<from gen_assets>"
for abi in arm64-v8a armeabi-v7a; do
  cargo run --bin patch_so -- \
    app/src/main/jniLibs/$abi/libcitadel.so \
    $FIRMWARE_SECRET
done
```

**6. Place assets in the APK**:

```
app/src/main/
├── assets/
│   ├── license.bin
│   ├── firmware.bin
│   └── codesign.bin
├── jniLibs/
│   ├── arm64-v8a/libcitadel.so
│   └── armeabi-v7a/libcitadel.so
└── java/.../SecureVm.kt
```

### Runtime API (Kotlin)

```kotlin
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        SecureVm().use { vm ->
            // Verify all assets and start the VM
            when (vm.startFromAssets(applicationContext)) {
                SecureVm.START_OK          -> { /* proceed */ }
                SecureVm.ERROR_ENVIRONMENT -> throw SecurityException("hostile environment")
                SecureVm.ERROR_INTEGRITY   -> throw SecurityException("assets tampered")
                SecureVm.ERROR_LICENSE     -> throw SecurityException("licence invalid")
                SecureVm.ERROR_FIRMWARE    -> throw SecurityException("firmware corrupt")
            }

            // Execute firmware — returns top-of-stack as Long
            val result: Long = vm.run()

            // Encrypt sensitive data using the customer key (key never exposed to Kotlin)
            val encrypted: ByteArray? = vm.encryptData("api_token".toByteArray())

            // Store a secret, hardware-backed when available
            vm.storeSecret("oauth_refresh", tokenBytes, passphrase)
            val token = vm.loadSecret("oauth_refresh", passphrase)
        }
        // SecureVm.close() zeros all key material
    }

    override fun onStop() {
        super.onStop()
        vm.stop()  // clear session key while app is backgrounded
    }
}
```

The `SecureVm` handle is opaque from Kotlin's perspective. All cryptographic operations happen inside the Rust layer via JNI. The raw key never crosses the JNI boundary — only ciphertexts and plaintexts do.

### Adding a New Licence (Multi-Tenant)

Each customer or distribution channel gets its own `licensepack.json` and therefore its own `license.bin` with a unique `firmware_secret` and `opcode_seed`. The same firmware source can produce firmware encrypted differently for each licence — the opcodes are remapped per seed, so even the same logic produces different bytecode for each customer.

```
Customer A licence                Customer B licence
  opcode_seed = 0xAABB...          opcode_seed = 0x1122...
  firmware_secret = 0xDEAD...      firmware_secret = 0xBEEF...
       │                                 │
  firmware_A.bin                   firmware_B.bin
  (identical logic,                (identical logic,
   different ciphertext,            different ciphertext,
   different opcode layout)         different opcode layout)
```

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Native runtime | Rust (stable, `no_std`-compatible core) |
| JNI bridge | `jni 0.21` crate |
| Symmetric encryption | AES-256-GCM (`aes-gcm 0.10`) |
| Key derivation | Argon2id (`argon2 0.5`) |
| Code signing | Ed25519 (`ed25519-dalek 2.0`) |
| Hashing / MAC | SHA-256, HMAC-SHA-256 (`sha2`, `hmac`) |
| String obfuscation | `obfstr 0.4` (compile-time XOR) |
| White-box AES | Chow-style T-tables (custom implementation) |
| Memory safety | `zeroize 1.0`, `subtle 2.0` (constant-time ops) |
| APK parsing | `zip 0.6` (cert extraction from META-INF) |
| Android Keystore | JNI calls to `KeyGenerator` / `Cipher` APIs |
| Control-flow obfuscation (Codex) | LLVM CFF + SUB passes (nightly Rust) |
| Target ABIs | `arm64-v8a`, `armeabi-v7a`, `x86_64-android` |

---

## Threat Model

| Threat | Mitigation | Strength |
|--------|-----------|----------|
| Extract firmware from APK | AES-256-GCM; key derived from app identity | Very High |
| Transplant assets to different app | License key KDF uses cert hash + package ID | Very High |
| Re-sign APK with attacker key | Cert hash changes → Argon2id produces wrong key | Very High |
| Offline brute-force license key | Argon2id (64 MB, 3 iterations) + vendor secret | High |
| Patch the native `.so` | SHA-256 slot (early) + HMAC slot (keyed, late) | High |
| Attach Frida / Xposed | 6 debugger checks, periodic re-check during execution | High |
| Run on rooted device | 7 root checks | Medium |
| Run on emulator | 5 emulator checks | Medium |
| Bypass opcode analysis | Per-license Fisher-Yates opcode shuffle | Medium |
| Customer key from memory dump | WBC T-tables or hardware Keystore | Medium–High |
| Physical / DCA / fault injection | Out of scope (hardware Keystore partially mitigates) | N/A |

---

## What Citadel Is Not

Citadel is a strong deterrent and a significant engineering challenge for an attacker — but it is not magic. A nation-state adversary with physical access, cold-boot equipment, and differential fault analysis capability is outside the threat model. The design goal is to make attacking a Citadel-protected app more expensive than the value of the protected logic, for the realistic attacker population (reverse engineers, script kiddies, casual pirates).

The defence-in-depth principle means each layer buys time even if the ones below it fail. An attacker who bypasses the environment checks still faces encrypted firmware. One who extracts the license ciphertext still needs to brute-force Argon2id against a vendor-held secret. One who patches the `.so` hits the HMAC wall. The layers are designed so that bypassing any one of them does not open a straight path to the firmware.

---

## What's Next

A few directions we are exploring:

- **Post-quantum signing** — migrating from Ed25519 to CRYSTALS-Dilithium for the codesign step
- **Attestation integration** — binding to Android's Play Integrity API verdict as an additional identity factor
- **Hardware-backed firmware keys** — importing `firmware_key` into the TEE rather than deriving it in software
- **Bytecode complexity** — extending the instruction set with floating-point and bitfield operations for richer firmware programs

---

Citadel is written in Rust, runs entirely in native code, and adds roughly 2–3 MB to an APK. For applications where the business logic is the product, that overhead is the smallest price on the table.
