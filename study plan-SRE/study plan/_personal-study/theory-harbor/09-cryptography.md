# 09 — Cryptography: AES-256-GCM

> **Goal:** Understand why credentials must be encrypted, how AES-256-GCM works, what a nonce is, and how this system encrypts Harbor passwords at rest.

---

## 1. The Problem: Credentials at Rest

After Harbor is deployed, its admin password and robot token must be stored somewhere so users can retrieve them later. Where?

```
Option 1: Store in plain text in PostgreSQL
  password = "xK9!mP2@qR5"
  Problem: anyone who can read the DB (DB admin, leaked backup, SQL injection)
           immediately has Harbor credentials for ALL tenants

Option 2: Hash with bcrypt
  stored = bcrypt("xK9!mP2@qR5")
  Problem: hashing is ONE-WAY — you can verify a password but can't recover it.
           We need to RETURN the password to the user.

Option 3: Encrypt with AES-256-GCM
  stored = AES_encrypt("xK9!mP2@qR5", masterKey)
  User requests credentials → server decrypts and returns
  If DB is leaked without the master key → data is useless
  ✓ This is what the system does
```

---

## 2. Symmetric vs Asymmetric Encryption

```
SYMMETRIC (one key for both encrypt and decrypt)
  Key:  🗝️
  Encrypt: plaintext + 🗝️ → ciphertext
  Decrypt: ciphertext + 🗝️ → plaintext

  Examples: AES, ChaCha20
  Use case: encrypting data at rest (fast, efficient)
  Risk: key must be kept secret; everyone who encrypts also can decrypt

ASYMMETRIC (public key encrypts, private key decrypts)
  Keys: 🔓 public key, 🔒 private key
  Encrypt: plaintext + 🔓 → ciphertext
  Decrypt: ciphertext + 🔒 → plaintext

  Examples: RSA, ECDH
  Use case: key exchange, JWT signing
  Risk: slower; private key must be protected
```

This system uses **symmetric AES** to encrypt credentials at rest.

---

## 3. AES — Advanced Encryption Standard

AES is a **block cipher** — it encrypts data in fixed 128-bit (16-byte) blocks.

```
Plaintext: "supersecretpassword"
               │
               ▼
AES block cipher (operates on 16-byte blocks)
               │
               ▼
Ciphertext: \x8f\x3a\x7b...  (looks like random bytes)
```

Key sizes:
- AES-128: 128-bit key (16 bytes)
- AES-192: 192-bit key (24 bytes)
- AES-256: **256-bit key (32 bytes)** ← this system uses this

AES-256 has `2^256` possible keys. Even checking a trillion keys per second would take longer than the age of the universe.

---

## 4. GCM Mode — Authenticated Encryption

AES alone (in ECB or CBC mode) has problems:

```
ECB mode PROBLEM:
  "AAAA" → \x8f\x3a    "AAAA" → \x8f\x3a    (same input = same output!)
  An attacker can see patterns even without decrypting.

CBC mode PROBLEM:
  Encrypts data but doesn't detect if someone tampered with the ciphertext.
  "Bit flipping attack" — attacker modifies ciphertext → decrypts to different plaintext.
```

**GCM (Galois/Counter Mode)** solves both problems:

```
AES-256-GCM provides:
  1. Confidentiality:   ciphertext looks random
  2. Authentication:    MAC tag detects any tampering
  3. Uses a Nonce:      same plaintext + different nonce = different ciphertext
```

GCM is called **AEAD** (Authenticated Encryption with Associated Data).

---

## 5. The Nonce — Why It's Critical

A **nonce** (Number used ONCE) is a random value that ensures:
- The same plaintext encrypted twice produces **different ciphertext** each time
- Without the correct nonce, you can't decrypt

```
Encryption:
  nonce = random 12 bytes (e.g.: \x3f\x7a\x2c...)
  ciphertext = AES_GCM_encrypt(plaintext, key, nonce)

Decryption:
  plaintext = AES_GCM_decrypt(ciphertext, key, nonce)
  ← requires the same nonce that was used to encrypt!
```

**Critical rule:** Never reuse a nonce with the same key. If you encrypt two different messages with the same key+nonce, an attacker can recover both plaintexts. This is why the code generates a fresh random nonce for every encryption.

---

## 6. The Implementation

```go
// From backend/internal/crypto/aes.go

type Cipher struct {
    aesGCM cipher.AEAD   // AEAD = Authenticated Encryption with Associated Data
}

func NewCipher(rawKey []byte) (*Cipher, error) {
    // Support both raw 32 bytes and hex-encoded string (from openssl rand -hex 32)
    key := rawKey
    trimmed := strings.TrimSpace(string(rawKey))
    if len(trimmed) == 64 {             // 64 hex chars = 32 bytes
        decoded, _ := hex.DecodeString(trimmed)
        key = decoded
    }

    if len(key) != 32 {
        return nil, fmt.Errorf("master key must be 32 bytes (got %d)", len(key))
    }

    // Create the AES block cipher
    block, _ := aes.NewCipher(key)

    // Wrap it in GCM mode
    gcm, _ := cipher.NewGCM(block)

    return &Cipher{aesGCM: gcm}, nil
}
```

### Encrypt

```go
func (c *Cipher) Encrypt(plaintext []byte) (ciphertext, nonce []byte, err error) {
    // 1. Generate a fresh random nonce (12 bytes for GCM)
    nonce = make([]byte, c.aesGCM.NonceSize())  // 12 bytes
    io.ReadFull(rand.Reader, nonce)              // ← crypto/rand, NOT math/rand!

    // 2. Encrypt: Seal(dst, nonce, plaintext, additionalData)
    // Returns: ciphertext + 16-byte authentication tag appended
    ciphertext = c.aesGCM.Seal(nil, nonce, plaintext, nil)

    return ciphertext, nonce, nil  // caller MUST store both
}
```

```
Before encryption:
  plaintext = "xK9!mP2@qR5#nL8$"  (Harbor admin password)
  nonce     = random 12 bytes

After encryption:
  ciphertext = <encrypted bytes + 16-byte auth tag>
  nonce      = <the random 12 bytes>

In database (registry_credentials table):
  encrypted_admin_pw = <ciphertext>   BYTEA column
  admin_pw_nonce     = <nonce>        BYTEA column
```

### Decrypt

```go
func (c *Cipher) Decrypt(ciphertext, nonce []byte) ([]byte, error) {
    if len(nonce) != c.aesGCM.NonceSize() {
        return nil, errors.New("invalid nonce size")
    }

    // Open = verify auth tag + decrypt
    plaintext, err := c.aesGCM.Open(nil, nonce, ciphertext, nil)
    if err != nil {
        // Decryption fails if:
        // 1. Wrong key
        // 2. Wrong nonce
        // 3. Ciphertext was tampered with (auth tag mismatch!)
        return nil, fmt.Errorf("decrypt: %w (ciphertext may be tampered)", err)
    }
    return plaintext, nil
}
```

---

## 7. The Master Key

The master key is a 32-byte (256-bit) random value. It **never touches the database**:

```
Generation (one time, during cluster setup):
  openssl rand -hex 32 > master-key
  # produces: a3f7c8d2...  (64 hex chars = 32 bytes)

Storage:
  kubectl create secret generic registry-master-key \
    --from-file=master-key=./master-key \
    --namespace registry-controller

Pod mounts:
  volumes:
    - name: master-key
      secret:
        secretName: registry-master-key
        defaultMode: 0400          # read-only, owner only

In container:
  /etc/secrets/master-key   ← file with the key bytes
```

For local dev, `dev/master-key` is pre-generated and mounted:

```yaml
# docker-compose.dev.yml
volumes:
  - ./dev/master-key:/etc/secrets/master-key:ro
```

---

## 8. The Full Encrypt → Store → Retrieve Flow

### During Deployment (Worker Step 6)

```go
// From backend/internal/worker/deploy_worker.go

// Encrypt the robot token
encToken, tokenNonce, _ := w.cipher.EncryptString(robot.Secret)
//          ^ciphertext   ^nonce

// Encrypt the admin password
encAdmin, adminNonce, _ := w.cipher.EncryptString(adminPass)

// Store both in DB (plaintext NEVER written to DB)
w.store.SaveCredentials(ctx, &db.RegistryCredentials{
    TenantID:         tenantID,
    RobotUsername:    robot.Name,
    EncryptedToken:   encToken,    // ciphertext bytes
    TokenNonce:       tokenNonce,  // nonce bytes
    AdminUsername:    "admin",
    EncryptedAdminPW: encAdmin,
    AdminPWNonce:     adminNonce,
})
```

### When User Requests Credentials (Handler)

```go
// In the credentials handler (conceptual)
creds, _ := h.store.GetCredentials(ctx, tenantID)

// Decrypt on the fly — plaintext only exists in memory, never on disk
robotToken, _ := h.cipher.DecryptString(creds.EncryptedToken, creds.TokenNonce)
adminPass, _  := h.cipher.DecryptString(creds.EncryptedAdminPW, creds.AdminPWNonce)

// Return to the authenticated user
c.JSON(200, gin.H{
    "robotUsername": creds.RobotUsername,
    "robotPassword": robotToken,    // ← decrypted, returned over HTTPS
    "adminUsername": "admin",
    "adminPassword": adminPass,
})
```

```
DB contents (what an attacker sees if they steal the DB):
  robot_username:     "robot$ci-default"           ← not sensitive
  encrypted_token:    \x8f\x3a\x7b\x2c...         ← useless without master key
  token_nonce:        \x3f\x7a\x2c...              ← useless without master key
  encrypted_admin_pw: \xa1\x4d\x8e...              ← useless without master key
  admin_pw_nonce:     \x91\x2b\x5f...              ← useless without master key

Without the master key file: attackers have NOTHING.
The master key is in a Kubernetes Secret, separate from the DB.
```

---

## 9. Password Generation

The system generates random passwords for Harbor admin and PostgreSQL using:

```go
func GeneratePassword(length int) (string, error) {
    const charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*"
    b := make([]byte, length)
    io.ReadFull(rand.Reader, b)   // ← crypto/rand (NOT math/rand!)

    for i := range b {
        b[i] = charset[int(b[i])%len(charset)]  // map random byte to charset
    }
    return string(b), nil
}
```

**`crypto/rand` vs `math/rand`:**

```
math/rand:
  Seeded with a predictable number
  If attacker knows the seed → can predict all "random" values
  NEVER use for security-sensitive randomness

crypto/rand:
  Reads from OS entropy pool (/dev/urandom)
  Truly unpredictable
  Always use for keys, nonces, passwords, tokens
```

---

## 🏋️ Exercises

### Exercise 1 — Generate a Master Key
```bash
# Generate a 32-byte hex key (what openssl rand -hex 32 does)
openssl rand -hex 32

# The output is 64 hex characters = 32 bytes
# How does the code detect that the key is hex-encoded?
# Look at NewCipher() in crypto/aes.go — find the len(trimmed) == 64 check
```

### Exercise 2 — Manually Encrypt Something in Go
Write a tiny Go program (or trace through the code):
```go
key, _ := hex.DecodeString("a3f7c8d2...") // use a real 32-byte hex key
cipher, _ := crypto.NewCipher(key)

ciphertext, nonce, _ := cipher.EncryptString("my-secret-password")
fmt.Printf("ciphertext: %x\n", ciphertext)
fmt.Printf("nonce:      %x\n", nonce)

// Now decrypt:
plaintext, _ := cipher.DecryptString(ciphertext, nonce)
fmt.Println(plaintext)   // "my-secret-password"
```

### Exercise 3 — What Happens With Wrong Nonce?
```go
cipher.EncryptString("password") → ciphertext, nonce

// Try decrypting with a different nonce:
wrongNonce := make([]byte, 12) // all zeros
cipher.Decrypt(ciphertext, wrongNonce)
// What error do you get?
```

### Exercise 4 — Threat Model
Consider these attack scenarios. For each, does the encryption protect the credentials?

| Attack | Protected? |
|--------|-----------|
| Attacker gets a copy of the PostgreSQL database dump | ? |
| Attacker gets the master-key Kubernetes Secret but not the DB | ? |
| Attacker gets both the DB dump AND the master-key | ? |
| Attacker intercepts HTTPS traffic between user and API | ? |
| Attacker gets shell access to the provisioner pod | ? |

### Exercise 5 — Read the AEAD Seal/Open API
The `cipher.AEAD` interface has `Seal` and `Open`. Look up the Go standard library docs:
- What does `Seal(dst, nonce, plaintext, additionalData []byte)` return?
- What is `additionalData` and why is it `nil` here?
- What does `Open` return if the authentication tag doesn't match?

---

## Summary

| Concept | What It Is |
|---------|-----------|
| **AES-256** | Symmetric block cipher with a 256-bit key |
| **GCM mode** | Makes AES authenticated — detects tampering |
| **AEAD** | Authenticated Encryption with Associated Data |
| **Nonce** | 12-byte random value — must be unique per encryption |
| **Auth tag** | 16-byte value appended to ciphertext — proves integrity |
| **Master key** | 32-byte key in K8s Secret — never in DB |
| **Encrypt** | plaintext + key + nonce → ciphertext |
| **Decrypt** | ciphertext + key + nonce → plaintext (or error if tampered) |
| **crypto/rand** | OS entropy pool — cryptographically secure randomness |
| **BYTEA** | PostgreSQL type for storing raw binary (ciphertext, nonces) |

**Next:** [10 — Async Workers & State Machines →](./10-async-workers-state-machines.md)
