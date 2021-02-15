---
layout: post
title: "Generating common test certificates"
---

```
#
# Generates common certificates for testing usages.
# The generated certificates cover the following combinations.
#
# +==============+=============+==================+=========+
# |  Public Key  |  Signature  |  Key size/Curve  |  Hash   |
# +==============+=============+==================+=========+
# |  RSA         |  RSA        |  2048            |  SHA256 |
# +--------------+-------------+------------------+---------+
# |  RSA         |  RSA        |  1024            |  SHA1   |
# +--------------+-------------+------------------+---------+
# |  EC          |  ECDSA      |  SECP256R1       |  SHA256 |
# +--------------+-------------+------------------+---------+
# |  EC          |  ECDSA      |  SECP384R1       |  SHA384 |
# +--------------+-------------+------------------+---------+
# |  EC          |  ECDSA      |  SECP521R1       |  SHA512 |
# +--------------+-------------+------------------+---------+
# |  EC          |  ECDSA      |  SECP256R1       |  SHA1   |
# +--------------+-------------+------------------+---------+
# |  EC          |  RSA        |  2048            |  SHA256 |
# +--------------+-------------+------------------+---------+
# |  EC          |  RSA        |  1024            |  SHA1   |
# +--------------+-------------+------------------+---------+
# |  RSASSA-PSS  |  RSASSA-PSS |  1024            |  SHA256 |
# +--------------+-------------+------------------+---------+
# |  DSA         |  DSA        |  2048            |  SHA256 |
# +--------------+-------------+------------------+---------+
# |  DSA         |  DSA        |  1024            |  SHA1   |
# +--------------+-------------+------------------+---------+
#

#!/bin/bash

echo "Generate X.509 version 3 extensions for CA"
cat > ca.ext << EOF
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
basicConstraints=critical,CA:TRUE
keyUsage=critical,keyCertSign,cRLSign,digitalSignature
extendedKeyUsage=critical,OCSPSigning
EOF

echo "Generate X.509 version 3 extensions for EE"
cat > ee.ext << EOF
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
EOF

####################

echo "CA, SHA256withRSA, 2048 bits"
openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out ca-rsa-2048-sha256.key
openssl req -new -key ca-rsa-2048-sha256.key -subj "/CN=ca-rsa-2048-sha256" -sha256 -out ca-rsa-2048-sha256.csr
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in ca-rsa-2048-sha256.csr -sha256 -signkey ca-rsa-2048-sha256.key -out ca-rsa-2048-sha256.cer

echo "server (localhost), SHA256withRSA, 2048 bits"
openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out localhost-rsa-2048-sha256.key
openssl req -new -key localhost-rsa-2048-sha256.key -subj "/CN=localhost" -sha256 -out localhost-rsa-2048-sha256.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in localhost-rsa-2048-sha256.csr -sha256 -CA ca-rsa-2048-sha256.cer -CAkey ca-rsa-2048-sha256.key -out localhost-rsa-2048-sha256.cer

echo "client, SHA256withRSA, 2048 bits"
openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out client-rsa-2048-sha256.key
openssl req -new -key client-rsa-2048-sha256.key -subj "/CN=client-rsa-2048-sha256" -sha256 -out client-rsa-2048-sha256.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in client-rsa-2048-sha256.csr -sha256 -CA ca-rsa-2048-sha256.cer -CAkey ca-rsa-2048-sha256.key -out client-rsa-2048-sha256.cer

####################

echo "CA, SHA1withRSA, 1024 bits"
openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:1024 -pkeyopt rsa_keygen_pubexp:65537 -out ca-rsa-1024-sha1.key
openssl req -new -key ca-rsa-1024-sha1.key -subj "/CN=ca-rsa-1024-sha1" -sha1 -out ca-rsa-1024-sha1.csr
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in ca-rsa-1024-sha1.csr -sha1 -signkey ca-rsa-1024-sha1.key -out ca-rsa-1024-sha1.cer

echo "server (localhost), SHA1withRSA, 1024 bits"
openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:1024 -pkeyopt rsa_keygen_pubexp:65537 -out localhost-rsa-1024-sha1.key
openssl req -new -key localhost-rsa-1024-sha1.key -subj "/CN=localhost" -sha1 -out localhost-rsa-1024-sha1.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in localhost-rsa-1024-sha1.csr -sha1 -CA ca-rsa-1024-sha1.cer -CAkey ca-rsa-1024-sha1.key -out localhost-rsa-1024-sha1.cer

echo "client, SHA1withRSA, 1024 bits"
openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:1024 -pkeyopt rsa_keygen_pubexp:65537 -out client-rsa-1024-sha1.key
openssl req -new -key client-rsa-1024-sha1.key -subj "/CN=client-rsa-1024-sha1" -sha1 -out client-rsa-1024-sha1.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in client-rsa-1024-sha1.csr -sha1 -CA ca-rsa-1024-sha1.cer -CAkey ca-rsa-1024-sha1.key -out client-rsa-1024-sha1.cer

####################

echo "CA, SHA256withECDSA, secp256r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -pkeyopt ec_param_enc:named_curve -out ca-ec-secp256r1-sha256.key
openssl req -new -key ca-ec-secp256r1-sha256.key -subj "/CN=ca-ec-secp256r1-sha256" -sha256 -out ca-ec-secp256r1-sha256.csr
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in ca-ec-secp256r1-sha256.csr -sha256 -signkey ca-ec-secp256r1-sha256.key -out ca-ec-secp256r1-sha256.cer

echo "server (localhost), SHA256withECDSA, secp256r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -pkeyopt ec_param_enc:named_curve -out localhost-ec-secp256r1-sha256.key
openssl req -new -key localhost-ec-secp256r1-sha256.key -subj "/CN=localhost" -sha256 -out localhost-ec-secp256r1-sha256.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in localhost-ec-secp256r1-sha256.csr -sha256 -CA ca-ec-secp256r1-sha256.cer -CAkey ca-ec-secp256r1-sha256.key -out localhost-ec-secp256r1-sha256.cer

echo "client, SHA256withECDSA, secp256r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -pkeyopt ec_param_enc:named_curve -out client-ec-secp256r1-sha256.key
openssl req -new -key client-ec-secp256r1-sha256.key -subj "/CN=client-ec-secp256r1-sha256" -sha256 -out client-ec-secp256r1-sha256.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in client-ec-secp256r1-sha256.csr -sha256 -CA ca-ec-secp256r1-sha256.cer -CAkey ca-ec-secp256r1-sha256.key -out client-ec-secp256r1-sha256.cer

####################

echo "CA, SHA384withECDSA, secp384r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-384 -pkeyopt ec_param_enc:named_curve -out ca-ec-secp384r1-sha384.key
openssl req -new -key ca-ec-secp384r1-sha384.key -subj "/CN=ca-ec-secp384r1-sha384" -sha384 -out ca-ec-secp384r1-sha384.csr
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in ca-ec-secp384r1-sha384.csr -sha384 -signkey ca-ec-secp384r1-sha384.key -out ca-ec-secp384r1-sha384.cer

echo "server (localhost), SHA384withECDSA, secp384r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-384 -pkeyopt ec_param_enc:named_curve -out localhost-ec-secp384r1-sha384.key
openssl req -new -key localhost-ec-secp384r1-sha384.key -subj "/CN=localhost" -sha384 -out localhost-ec-secp384r1-sha384.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in localhost-ec-secp384r1-sha384.csr -sha384 -CA ca-ec-secp384r1-sha384.cer -CAkey ca-ec-secp384r1-sha384.key -out localhost-ec-secp384r1-sha384.cer

echo "client, SHA384withECDSA, secp384r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-384 -pkeyopt ec_param_enc:named_curve -out client-ec-secp384r1-sha384.key
openssl req -new -key client-ec-secp384r1-sha384.key -subj "/CN=client-ec-secp384r1-sha384" -sha384 -out client-ec-secp384r1-sha384.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in client-ec-secp384r1-sha384.csr -sha384 -CA ca-ec-secp384r1-sha384.cer -CAkey ca-ec-secp384r1-sha384.key -out client-ec-secp384r1-sha384.cer

####################

echo "CA, SHA512withECDSA, secp521r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-521 -pkeyopt ec_param_enc:named_curve -out ca-ec-secp521r1-sha512.key
openssl req -new -key ca-ec-secp521r1-sha512.key -subj "/CN=ca-ec-secp521r1-sha512" -sha512 -out ca-ec-secp521r1-sha512.csr
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in ca-ec-secp521r1-sha512.csr -sha512 -signkey ca-ec-secp521r1-sha512.key -out ca-ec-secp521r1-sha512.cer

echo "server (localhost), SHA512withECDSA, secp521r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-521 -pkeyopt ec_param_enc:named_curve -out localhost-ec-secp521r1-sha512.key
openssl req -new -key localhost-ec-secp521r1-sha512.key -subj "/CN=localhost" -sha512 -out localhost-ec-secp521r1-sha512.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in localhost-ec-secp521r1-sha512.csr -sha512 -CA ca-ec-secp521r1-sha512.cer -CAkey ca-ec-secp521r1-sha512.key -out localhost-ec-secp521r1-sha512.cer

echo "client, SHA512withECDSA, secp521r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-521 -pkeyopt ec_param_enc:named_curve -out client-ec-secp521r1-sha512.key
openssl req -new -key client-ec-secp521r1-sha512.key -subj "/CN=client-ec-secp521r1-sha512" -sha512 -out client-ec-secp521r1-sha512.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in client-ec-secp521r1-sha512.csr -sha512 -CA ca-ec-secp521r1-sha512.cer -CAkey ca-ec-secp521r1-sha512.key -out client-ec-secp521r1-sha512.cer

####################

echo "CA, SHA1withECDSA, secp256r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -pkeyopt ec_param_enc:named_curve -out ca-ec-secp256r1-sha1.key
openssl req -new -key ca-ec-secp256r1-sha1.key -subj "/CN=ca-ec-secp256r1-sha1" -sha1 -out ca-ec-secp256r1-sha1.csr
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in ca-ec-secp256r1-sha1.csr -sha1 -signkey ca-ec-secp256r1-sha1.key -out ca-ec-secp256r1-sha1.cer

echo "server (localhost), SHA1withECDSA, secp256r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -pkeyopt ec_param_enc:named_curve -out localhost-ec-secp256r1-sha1.key
openssl req -new -key localhost-ec-secp256r1-sha1.key -subj "/CN=localhost" -sha1 -out localhost-ec-secp256r1-sha1.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in localhost-ec-secp256r1-sha1.csr -sha1 -CA ca-ec-secp256r1-sha1.cer -CAkey ca-ec-secp256r1-sha1.key -out localhost-ec-secp256r1-sha1.cer

echo "client, SHA1withECDSA, secp256r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -pkeyopt ec_param_enc:named_curve -out client-ec-secp256r1-sha1.key
openssl req -new -key client-ec-secp256r1-sha1.key -subj "/CN=client-ec-secp256r1-sha1" -sha1 -out client-ec-secp256r1-sha1.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in client-ec-secp256r1-sha1.csr -sha1 -CA ca-ec-secp256r1-sha1.cer -CAkey ca-ec-secp256r1-sha1.key -out client-ec-secp256r1-sha1.cer

####################

echo "CA, SHA256withRSA, 2048 bits"
openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out ca-ecrsa-2048-sha256.key
openssl req -new -key ca-ecrsa-2048-sha256.key -subj "/CN=ca-ecrsa-2048-sha256" -sha256 -out ca-ecrsa-2048-sha256.csr
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in ca-ecrsa-2048-sha256.csr -sha256 -signkey ca-ecrsa-2048-sha256.key -out ca-ecrsa-2048-sha256.cer

echo "server (localhost), SHA256withRSA, secp256r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -pkeyopt ec_param_enc:named_curve -out localhost-ecrsa-secp256r1-sha256.key
openssl req -new -key localhost-ecrsa-secp256r1-sha256.key -subj "/CN=localhost" -sha256 -out localhost-ecrsa-secp256r1-sha256.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in localhost-ecrsa-secp256r1-sha256.csr -sha256 -CA ca-ecrsa-2048-sha256.cer -CAkey ca-ecrsa-2048-sha256.key -out localhost-ecrsa-secp256r1-sha256.cer

echo "client, SHA256withRSA, secp256r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -pkeyopt ec_param_enc:named_curve -out client-ecrsa-secp256r1-sha256.key
openssl req -new -key client-ecrsa-secp256r1-sha256.key -subj "/CN=client-ecrsa-secp256r1-sha256" -sha256 -out client-ecrsa-secp256r1-sha256.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in client-ecrsa-secp256r1-sha256.csr -sha256 -CA ca-ecrsa-2048-sha256.cer -CAkey ca-ecrsa-2048-sha256.key -out client-ecrsa-secp256r1-sha256.cer

####################

echo "CA, SHA1withRSA, 1024 bits"
openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:1024 -pkeyopt rsa_keygen_pubexp:65537 -out ca-ecrsa-1024-sha1.key
openssl req -new -key ca-ecrsa-1024-sha1.key -subj "/CN=ca-ecrsa-1024-sha1" -sha1 -out ca-ecrsa-1024-sha1.csr
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in ca-ecrsa-1024-sha1.csr -sha1 -signkey ca-ecrsa-1024-sha1.key -out ca-ecrsa-1024-sha1.cer

echo "server (localhost), SHA1withRSA, secp256r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -pkeyopt ec_param_enc:named_curve -out localhost-ecrsa-secp256r1-sha1.key
openssl req -new -key localhost-ecrsa-secp256r1-sha1.key -subj "/CN=localhost" -sha1 -out localhost-ecrsa-secp256r1-sha1.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in localhost-ecrsa-secp256r1-sha1.csr -sha1 -CA ca-ecrsa-1024-sha1.cer -CAkey ca-ecrsa-1024-sha1.key -out localhost-ecrsa-secp256r1-sha1.cer

echo "client, SHA1withRSA, secp256r1"
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:P-256 -pkeyopt ec_param_enc:named_curve -out client-ecrsa-secp256r1-sha1.key
openssl req -new -key client-ecrsa-secp256r1-sha1.key -subj "/CN=client-ecrsa-secp256r1-sha1" -sha1 -out client-ecrsa-secp256r1-sha1.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in client-ecrsa-secp256r1-sha1.csr -sha1 -CA ca-ecrsa-1024-sha1.cer -CAkey ca-ecrsa-1024-sha1.key -out client-ecrsa-secp256r1-sha1.cer

####################

echo "CA, RSASSA-PSS, 2048 bits"
openssl genpkey -algorithm rsa-pss -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out ca-pss-2048-sha256.key
openssl req -new -key ca-pss-2048-sha256.key -subj "/CN=ca-pss-2048-sha256" -sha256 -out ca-pss-2048-sha256.csr
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in ca-pss-2048-sha256.csr -sha256 -signkey ca-pss-2048-sha256.key -out ca-pss-2048-sha256.cer

echo "server (localhost), RSASSA-PSS, 2048 bits"
openssl genpkey -algorithm rsa-pss -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out localhost-pss-2048-sha256.key
openssl req -new -key localhost-pss-2048-sha256.key -subj "/CN=localhost" -sha256 -out localhost-pss-2048-sha256.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in localhost-pss-2048-sha256.csr -sha256 -CA ca-pss-2048-sha256.cer -CAkey ca-pss-2048-sha256.key -out localhost-pss-2048-sha256.cer

echo "client, RSASSA-PSS, 2048 bits"
openssl genpkey -algorithm rsa-pss -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out client-pss-2048-sha256.key
openssl req -new -key client-pss-2048-sha256.key -subj "/CN=client-pss-2048-sha256" -sha256 -out client-pss-2048-sha256.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in client-pss-2048-sha256.csr -sha256 -CA ca-pss-2048-sha256.cer -CAkey ca-pss-2048-sha256.key -out client-pss-2048-sha256.cer

####################

echo "CA, SHA256withDSA, 2048 bits"
openssl genpkey -genparam -algorithm dsa -pkeyopt dsa_paramgen_bits:2048 -pkeyopt dsa_paramgen_q_bits:256 -out ca-dsa-2048-sha256.param
openssl genpkey -paramfile ca-dsa-2048-sha256.param -out ca-dsa-2048-sha256.key
openssl req -new -key ca-dsa-2048-sha256.key -subj "/CN=ca-dsa-2048-sha256" -sha256 -out ca-dsa-2048-sha256.csr
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in ca-dsa-2048-sha256.csr -sha256 -signkey ca-dsa-2048-sha256.key -out ca-dsa-2048-sha256.cer

echo "server (localhost), SHA256withDSA, 2048 bits"
openssl genpkey -genparam -algorithm dsa -pkeyopt dsa_paramgen_bits:2048 -pkeyopt dsa_paramgen_q_bits:256 -out localhost-dsa-2048-sha256.param
openssl genpkey -paramfile localhost-dsa-2048-sha256.param -out localhost-dsa-2048-sha256.key
openssl req -new -key localhost-dsa-2048-sha256.key -subj "/CN=localhost" -sha256 -out localhost-dsa-2048-sha256.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in localhost-dsa-2048-sha256.csr -sha256 -CA ca-dsa-2048-sha256.cer -CAkey ca-dsa-2048-sha256.key -out localhost-dsa-2048-sha256.cer

echo "client, SHA256withDSA, 2048 bits"
openssl genpkey -genparam -algorithm dsa -pkeyopt dsa_paramgen_bits:2048 -pkeyopt dsa_paramgen_q_bits:256 -out client-dsa-2048-sha256.param
openssl genpkey -paramfile client-dsa-2048-sha256.param -out client-dsa-2048-sha256.key
openssl req -new -key client-dsa-2048-sha256.key -subj "/CN=client-dsa-2048-sha256" -sha256 -out client-dsa-2048-sha256.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in client-dsa-2048-sha256.csr -sha256 -CA ca-dsa-2048-sha256.cer -CAkey ca-dsa-2048-sha256.key -out client-dsa-2048-sha256.cer

####################

echo "CA, SHA1withDSA, 1024 bits"
openssl genpkey -genparam -algorithm dsa -pkeyopt dsa_paramgen_bits:1024 -pkeyopt dsa_paramgen_q_bits:256 -out ca-dsa-1024-sha1.param
openssl genpkey -paramfile ca-dsa-1024-sha1.param -out ca-dsa-1024-sha1.key
openssl req -new -key ca-dsa-1024-sha1.key -subj "/CN=ca-dsa-1024-sha1" -sha1 -out ca-dsa-1024-sha1.csr
openssl x509 -extfile ca.ext -req -CAcreateserial -days 3650 -in ca-dsa-1024-sha1.csr -sha1 -signkey ca-dsa-1024-sha1.key -out ca-dsa-1024-sha1.cer

echo "server (localhost), SHA1withDSA, 1024 bits"
openssl genpkey -genparam -algorithm dsa -pkeyopt dsa_paramgen_bits:1024 -pkeyopt dsa_paramgen_q_bits:256 -out localhost-dsa-1024-sha1.param
openssl genpkey -paramfile localhost-dsa-1024-sha1.param -out localhost-dsa-1024-sha1.key
openssl req -new -key localhost-dsa-1024-sha1.key -subj "/CN=localhost" -sha1 -out localhost-dsa-1024-sha1.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in localhost-dsa-1024-sha1.csr -sha1 -CA ca-dsa-1024-sha1.cer -CAkey ca-dsa-1024-sha1.key -out localhost-dsa-1024-sha1.cer

echo "client, SHA1withDSA, 1024 bits"
openssl genpkey -genparam -algorithm dsa -pkeyopt dsa_paramgen_bits:1024 -pkeyopt dsa_paramgen_q_bits:256 -out client-dsa-1024-sha1.param
openssl genpkey -paramfile client-dsa-1024-sha1.param -out client-dsa-1024-sha1.key
openssl req -new -key client-dsa-1024-sha1.key -subj "/CN=client-dsa-1024-sha1" -sha1 -out client-dsa-1024-sha1.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -days 3650 -in client-dsa-1024-sha1.csr -sha1 -CA ca-dsa-1024-sha1.cer -CAkey ca-dsa-1024-sha1.key -out client-dsa-1024-sha1.cer

rm *.csr *.srl *.param
```

