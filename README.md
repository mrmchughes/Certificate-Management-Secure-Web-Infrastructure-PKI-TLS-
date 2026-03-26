# Certificate Management & Secure Web Infrastructure (PKI/TLS)

Built a complete **Public Key Infrastructure (PKI)** chain from a 
pre-existing Root CA through to a deployed, trust-verified TLS 
certificate on an Apache web server. Covers the full certificate 
lifecycle: key generation, CSR creation, CA signing with extensions, 
Apache HTTPS configuration, and trust chain validation.

## Overview

This lab simulates the role of a system administrator issuing and 
deploying a server certificate within an established PKI hierarchy. 
Rather than using a self-signed certificate, the server certificate 
is signed by an existing Root CA — modeling real enterprise PKI 
where a central CA governs certificate issuance.

**Core concepts demonstrated:**
- Root CA inspection (subject, issuer, validity, extensions)
- `CA:TRUE` vs `CA:FALSE` Basic Constraints and their enforcement
- RSA private key generation and permission hardening
- Certificate Signing Request (CSR) creation with subject metadata
- CA-signed certificate issuance with explicit X.509 v3 extensions
- `subjectAltName` (SAN) for modern TLS hostname validation
- `keyUsage` and `extendedKeyUsage` extension scoping
- Apache SSL/TLS configuration (`mod_ssl`, `default-ssl.conf`)
- Trust chain validation with and without CA bundle (`curl --cacert`)

## PKI Hierarchy
```
/ca/rootCA.crt  (CA:TRUE — trust anchor)
      │
      │ signs
      ▼
/certs/server.crt  (CA:FALSE — end-entity)
      │
      └── Deployed to Apache on port 443
```

## Implementation

### Part 1 — Root CA Inspection
```bash
# Inspect subject, issuer, validity window
openssl x509 -in /ca/rootCA.crt -noout -subject -issuer -dates

# Confirm CA:TRUE Basic Constraints
openssl x509 -in /ca/rootCA.crt -noout -text \
  | grep -A3 "Basic Constraints"
```

A Root CA must carry `CA:TRUE` so that clients recognize it as 
authorized to sign other certificates. Without this extension, 
the certificate cannot be used as a trust anchor — any cert it 
signs would fail chain validation.

### Part 2 — Server Private Key & CSR
```bash
# Generate RSA-2048 private key
openssl genrsa -out /certs/server.key 2048

# Harden permissions — private key never world-readable
chmod 600 /certs/server.key

# Generate CSR with subject metadata
openssl req -new -key /certs/server.key -out /certs/server.csr \
  -subj "/C=US/ST=MA/L=Boston/O=NEU/OU=hughes.michael/CN=localhost"

# Verify CSR contents
openssl req -in /certs/server.csr -noout -subject -text
```

In production PKI, a **Registration Authority (RA)** is the 
appropriate entity to collect and validate CSRs before forwarding 
them to the CA for signing — separating identity verification from 
certificate issuance.

### Part 3 — Sign with Root CA
```bash
# Extension file: server.ext
cat > /tmp/server.ext <<EOF
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = DNS:localhost,IP:127.0.0.1
EOF

# Sign CSR with Root CA
openssl x509 -req \
  -in /certs/server.csr \
  -CA /ca/rootCA.crt \
  -CAkey /ca/rootCA.key \
  -CAcreateserial \
  -out /certs/server.crt \
  -days 365 \
  -extfile /tmp/server.ext

# Verify extensions on issued cert
openssl x509 -in /certs/server.crt -noout -text \
  | grep -A3 "Basic Constraints"
```

`CN=localhost` is critical because TLS clients validate the 
server's hostname against the certificate's CN and SAN fields. 
A mismatch causes the connection to fail regardless of whether 
the certificate is validly signed — preventing a compromised 
certificate for one host from being reused on another.

### Part 4 — Apache HTTPS Configuration
```bash
sudo a2enmod ssl
sudo a2ensite default-ssl

# /etc/apache2/sites-available/default-ssl.conf
SSLCertificateFile    /certs/server.crt
SSLCertificateKeyFile /certs/server.key

sudo apachectl -k graceful
```

### Part 5 — Trust Chain Validation
```bash
# No CA bundle — curl rejects the self-signed chain
curl https://localhost
# → SSL certificate verify error (self-signed)

# Skip verification entirely — connection succeeds but untrusted
curl -k https://localhost
# → 200 OK (insecure, no chain validation)

# Explicit CA bundle — curl validates the full chain
curl https://localhost --cacert /ca/rootCA.crt
# → 200 OK (trusted, chain validated against Root CA)
```

The three behaviors illustrate the difference between an absent 
trust anchor (error), disabled validation (insecure bypass), and 
proper CA-anchored trust (correct production behavior).

## Files
```
/ca/
├── rootCA.crt          # Pre-existing Root CA certificate
└── rootCA.key          # Root CA private key (signing only)

/certs/
├── server.key          # Web server private key (mode 600)
├── server.csr          # Certificate Signing Request
└── server.crt          # CA-signed server certificate
```

## Key Takeaways

| Concept | Detail |
|---|---|
| `CA:TRUE` | Required for any cert that signs other certs |
| `CA:FALSE` | End-entity certs must not be usable as CAs |
| SAN extension | Modern TLS requires SAN — CN alone is deprecated |
| `keyUsage` scoping | Limits cert to intended operations only |
| `--cacert` flag | How clients establish trust without system-wide CA install |

## Course Context

Completed as part of **CY 5130 — Computer System Security**  
Northeastern University, Khoury College of Computer Sciences  
MSCY Align Program
