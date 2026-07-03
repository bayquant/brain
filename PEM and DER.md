---
tags: [security, cryptography, certificates, tls]
---
# PEM and DER

PEM and DER are the two core **encoding formats** for storing cryptographic keys, certificates, and certificate signing requests. They encode the same underlying ASN.1 data — PEM is just DER wrapped in Base64 text.

---

## CORE CONCEPTS

```
Raw data (private key, public key, certificate, CSR)
└── ASN.1 structure (defines the fields, per X.509 or PKCS standards)
    ├── DER — binary encoding of the ASN.1 structure
    └── PEM — Base64 encoding of the DER bytes, wrapped with header/footer lines
```

- **DER (Distinguished Encoding Rules)** — binary format, compact, not human-readable
- **PEM (Privacy Enhanced Mail)** — text format, Base64 + delimiters; the de facto standard for OpenSSL, Apache, nginx
- File extensions (`.pem`, `.crt`, `.cer`, `.key`) are conventions, not guarantees — the actual encoding must be checked

---

## PEM FORMAT

```
-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJAKL...
(base64-encoded DER bytes)
...s0Y1G6nR9V7B9=
-----END CERTIFICATE-----
```

Common block types found in `.pem` files:

```
-----BEGIN CERTIFICATE-----          # X.509 certificate
-----BEGIN CERTIFICATE REQUEST-----  # CSR
-----BEGIN PRIVATE KEY-----          # PKCS#8 private key
-----BEGIN RSA PRIVATE KEY-----      # PKCS#1 RSA private key (older)
-----BEGIN EC PRIVATE KEY-----       # EC private key
-----BEGIN PUBLIC KEY-----           # PKCS#8 public key
```

A single `.pem` file can contain multiple concatenated blocks — e.g. a certificate followed by its chain of intermediate CAs.

---

## RELATED FORMATS

| Format | Encoding | Contents | Typical extension |
|---|---|---|---|
| PEM         | Base64 text | key, cert, CSR, or chain | `.pem`, `.crt`, `.key` |
| DER         | binary      | single key or cert       | `.der`, `.cer` |
| PKCS#7      | binary/PEM  | cert chain, no private key | `.p7b`, `.p7c` |
| PKCS#12     | binary      | private key + cert + chain, password-protected | `.p12`, `.pfx` |

---

## OPENSSL CONVERSIONS

```bash
# PEM -> DER
openssl x509 -in cert.pem -outform DER -out cert.der

# DER -> PEM
openssl x509 -in cert.der -inform DER -outform PEM -out cert.pem

# PEM cert + key -> PKCS#12 (for import into browsers, Java keystores, etc.)
openssl pkcs12 -export -in cert.pem -inkey key.pem -out cert.p12

# PKCS#12 -> PEM (extract everything)
openssl pkcs12 -in cert.p12 -out cert.pem -nodes
```

---

## INSPECTING FILES

```bash
# View a PEM certificate's fields (subject, issuer, validity, SANs)
openssl x509 -in cert.pem -text -noout

# View a DER certificate (note -inform DER)
openssl x509 -in cert.der -inform DER -text -noout

# View a private key's details
openssl rsa -in key.pem -text -noout
openssl pkey -in key.pem -text -noout

# View a CSR
openssl req -in request.csr -text -noout

# Check a private key matches a certificate (moduli should match)
openssl x509 -noout -modulus -in cert.pem | openssl md5
openssl rsa -noout -modulus -in key.pem | openssl md5
```

---

## PKCS#1 VS PKCS#8 (PRIVATE KEYS)

Two different wrappers for private keys — a common source of "invalid key" errors.

```
-----BEGIN RSA PRIVATE KEY-----   # PKCS#1 — RSA-specific, older
-----BEGIN PRIVATE KEY-----       # PKCS#8 — algorithm-agnostic (RSA, EC, etc.), preferred
```

```bash
# Convert PKCS#1 -> PKCS#8
openssl pkcs8 -topk8 -nocrypt -in rsa_key.pem -out pkcs8_key.pem
```

---

## GENERATING KEYS AND CERTIFICATES

### PRIVATE KEYS

```bash
# RSA private key, PKCS#8, PEM (modern default)
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out key.pem

# RSA private key, PKCS#1, PEM (older, e.g. legacy nginx/Apache configs)
openssl genrsa -out key.pem 4096

# EC private key, PEM (smaller, faster than RSA at equivalent strength)
openssl ecparam -name prime256v1 -genkey -noout -out key.pem

# Encrypt the private key with a passphrase (AES-256)
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 \
  -aes-256-cbc -out key.pem
```

### CERTIFICATE SIGNING REQUEST (CSR)

A CSR is what you send to a Certificate Authority to get a signed certificate back.

```bash
# Generate a CSR from an existing private key
openssl req -new -key key.pem -out request.csr -subj "/CN=example.com"

# Generate a key + CSR in one step
openssl req -new -newkey rsa:4096 -nodes \
  -keyout key.pem -out request.csr -subj "/CN=example.com"

# CSR with Subject Alternative Names (SANs) — required by modern browsers
openssl req -new -key key.pem -out request.csr \
  -subj "/CN=example.com" \
  -addext "subjectAltName=DNS:example.com,DNS:www.example.com"
```

### SELF-SIGNED CERTIFICATE

Skips the CA — the cert signs itself. Fine for local dev/test, not trusted by browsers/clients by default.

```bash
# Key + self-signed cert in one step
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
  -days 365 -nodes -subj "/CN=localhost"

# Self-sign an existing CSR using an existing key
openssl x509 -req -in request.csr -signkey key.pem -out cert.pem -days 365

# Self-signed cert with SANs (needed for localhost testing in Chrome/Safari)
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
  -days 365 -nodes -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

### DIRECTLY IN DER

```bash
# Generate straight to DER instead of PEM
openssl req -x509 -newkey rsa:4096 -keyout key.der -out cert.der \
  -days 365 -nodes -subj "/CN=localhost" -outform DER
```

---

## PATTERNS

### BUILDING A FULL CHAIN FILE

Order matters: leaf certificate first, then intermediates, ending with (or omitting) the root.

```bash
cat leaf.pem intermediate.pem > fullchain.pem
```

### COMMON PITFALLS

- A `.crt`/`.cer` file might actually be DER, not PEM — check with `file cert.crt` or try opening it as text
- Mixing PKCS#1 and PKCS#8 key headers causes "unsupported" errors in some libraries (e.g. Node's `crypto`, some Java tools)
- PKCS#12 bundles are binary and password-protected; PEM bundles are plaintext — never commit either containing a private key to version control

See also [[Terraform]].
