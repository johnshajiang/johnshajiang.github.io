---
layout: post
title: "Generating certificates with OpenSSL"
---

This is a simple doc on generating certificates with OpenSSL.
It focus on three different certificate types, exactly the classic RSA and ECDSA and the relative new RSASSA-PSS.
It generates a CA and an end entity (EE) certificate for each type.
The content is straightforward and concise: Commands with comments.

Please note that the commands on different certificate types are quite similar.
Especially, the private key generation on different algorithms just uses tool `genpkey`, though some algorithms (e.g. `RSA`) have their own tool (e.g. `genrsa`).
This is deliberate. In further development, these commands could be abstracted as a single common certificate generation facility.

### OpenSSL configurations
```
# Generate X.509 version 3 extensions for CA
cat > ca.ext << EOF
basicConstraints=critical,CA:TRUE
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
keyUsage=critical,keyCertSign,cRLSign,digitalSignature
extendedKeyUsage=critical,codeSigning,timeStamping,OCSPSigning
EOF

# Generate X.509 version 3 extension file for EE
cat > ee.ext << EOF
basicConstraints=critical,CA:FALSE
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
EOF
```

### RSA certificates
```
# Generate RSA private key for RSA CA
# The key size is 2048; the exponent is 65537
openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out CA.key

# Generate certificate signing request for RSA CA
openssl req -new -key CA.key -subj "/CN=CA" -sha256 -out CA.csr

# Generate RSA CA based on the above CSR, and sign it with the above RSA CA key
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in CA.csr -sha256 -signkey CA.key -out CA.cer

# Generate RSA private key for RSA EE
openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out EE.key

# Generate certificate signing request for RSA EE
openssl req -new -key EE.key -subj "/CN=EE" -sha256 -out EE.csr

# Generate RSA EE based on the above CSR, and sign it with the above RSA CA
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in EE.csr -sha256 -CA CA.cer -CAkey CA.key -out EE.cer
```

### EC certificates
These commands and options are quit similar to those in section `RSA certificates`.
The main difference is the private key generation.

```
# Generate EC private key for EC CA
# The named curve is P-256 in NIST (or prime256v1 in ANSI X9.62, or secp256r1 in SECG)
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -pkeyopt ec_param_enc:named_curve -out CA.key

# Generate certificate signing request for EC CA
openssl req -new -key CA.key -subj "/CN=CA" -sha256 -out CA.csr

# Generate EC CA based on the above CSR, and sign it with the above EC CA key
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in CA.csr -sha256 -signkey CA.key -out CA.cer

# Generate EC private key for EC EE
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -pkeyopt ec_param_enc:named_curve -out EE.key

# Generate certificate signing request for EC EE
openssl req -new -key EE.key -subj "/CN=EE" -sha256 -out EE.csr

# Generate EC EE based on the above CSR, and sign it with the above EC CA
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in EE.csr -sha256 -CA CA.cer -CAkey CA.key -out EE.cer
```

### RSASSA-PSS certificates
These commands and options are almost the same as those in section `RSA certificates`.
The only difference is the public key algorithm, of course rsa-pss here.

```
# Generate RSASSA-PSS private key for RSASSA-PSS CA
# The key size is 2048; the exponent is 65537
openssl genpkey -algorithm rsa-pss -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out CA.key

# Generate certificate signing request for RSASSA-PSS CA
openssl req -new -key CA.key -subj "/CN=CA" -sha256 -out CA.csr

# Generate RSASSA-PSS CA based on the above CSR, and sign it with the above RSASSA-PSS CA key
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in CA.csr -sha256 -signkey CA.key -out CA.cer

# Generate RSASSA-PSS private key for RSASSA-PSS EE
openssl genpkey -algorithm rsa-pss -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out EE.key

# Generate certificate signing request for RSASSA-PSS EE
openssl req -new -key EE.key -subj "/CN=EE" -sha256 -out EE.csr

# Generate RSASSA-PSS EE based on the above CSR, and sign it with the above RSASSA-PSS CA
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in EE.csr -sha256 -CA CA.cer -CAkey CA.key -out EE.cer
```

### DSA certificates
These commands and options are quite similar to those in section `RSA certificates`.
The main difference is that it needs to generate key parameters before generating key.

```
# Generate DSA private key for DSA CA
openssl genpkey -genparam -algorithm dsa -pkeyopt dsa_paramgen_bits:2048 -pkeyopt dsa_paramgen_q_bits:256 -out CA.param
openssl genpkey -paramfile CA.param -out CA.key

# Generate certificate signing request for DSA CA
openssl req -new -key CA.key -subj "/CN=CA" -sha256 -out CA.csr

# Generate DSA CA based on the above CSR, and sign it with the above DSA CA key
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in CA.csr -sha256 -signkey CA.key -out CA.cer

# Generate DSA private key for DSA EE
openssl genpkey -genparam -algorithm dsa -pkeyopt dsa_paramgen_bits:2048 -pkeyopt dsa_paramgen_q_bits:256 -out EE.param
openssl genpkey -paramfile EE.param -out EE.key

# Generate certificate signing request for DSA EE
openssl req -new -key EE.key -subj "/CN=EE" -sha256 -out EE.csr

# Generate DSA EE based on the above CSR, and sign it with the above DSA CA
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in EE.csr -sha256 -CA CA.cer -CAkey CA.key -out EE.cer
```

