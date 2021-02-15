---
layout: post
title: "Check certificate revocation status with OpenSSL"
---

### Certificates
```
# X.509 certificate extension for CA
# The CA should have critical key usage crl signing and extended key usage OCSP signing.
cat > ca.ext << EOF
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
basicConstraints=critical,CA:TRUE
keyUsage=critical,keyCertSign,cRLSign,digitalSignature
extendedKeyUsage=critical,OCSPSigning
EOF

# X.509 certificate extension for end entity
cat > ee.ext << EOF
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
EOF

openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out ca.key
openssl req -new -key ca.key -subj "/CN=ca" -sha256 -out ca.csr
openssl x509 -extfile ca.ext -req -CAcreateserial -in ca.csr -sha256 -signkey ca.key -out ca.cer

openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out valid.key
openssl req -new -key valid.key -subj "/CN=valid" -sha256 -out valid.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -in valid.csr -sha256 -CA ca.cer -CAkey ca.key -out valid.cer

openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out revoked.key
openssl req -new -key revoked.key -subj "/CN=revoked" -sha256 -out revoked.csr
openssl x509 -extfile ee.ext -req -CAcreateserial -in revoked.csr -sha256 -CA ca.cer -CAkey ca.key -out revoked.cer
```

### CRL (Certificate Revocation List)
```
touch index

cat > openssl.cnf << EOF
[crl]
database = index
EOF

# Revoke certificate, and store the revocation status into index
openssl ca -config openssl.cnf -name crl -revoke revoked.cer -cert ca.cer -keyfile ca.key -crl_reason superseded -md sha256

# The status info for revoked.cer in index
cat index
R	200813034252Z	200714034259Z,superseded	06E1A7326485827C7D43F311CA03D1F968773FCD	unknown	/CN=revoked

# Generate CRL
openssl ca -config openssl.cnf -name crl -gencrl -cert ca.cer -keyfile ca.key -md sha256 -crldays 365 -out ca.crl

# Display CRL
openssl crl -text -in ca.crl
Certificate Revocation List (CRL):
        Version 2 (0x1)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = ca
        Last Update: Jul 14 03:42:59 2020 GMT
        Next Update: Jul 14 03:42:59 2021 GMT
Revoked Certificates:
    Serial Number: 06E1A7326485827C7D43F311CA03D1F968773FCD
        Revocation Date: Jul 14 03:42:59 2020 GMT
        CRL entry extensions:
            X509v3 CRL Reason Code: 
                Superseded
    Signature Algorithm: sha256WithRSAEncryption
         b3:40:0b:69:54:a2:80:4a:45:10:28:fe:26:89:e9:55:13:4f:
         c1:5c:6f:d4:88:42:49:ba:85:41:53:64:8a:25:d2:6f:03:14:
         21:56:39:d9:ef:5f:b9:23:ad:dd:90:a4:3e:84:32:bc:f2:76:
         db:f0:fa:eb:16:68:9e:89:62:6f:ac:50:84:e4:ad:b3:3d:fe:
         a6:6d:0a:2e:cf:ab:e9:2a:df:46:30:a5:38:7e:14:38:76:66:
         6e:17:dd:ec:f8:fc:af:66:82:2e:dc:06:43:cf:17:66:7a:2c:
         d8:6b:d4:98:19:06:81:6d:dd:5d:77:46:79:00:f4:c6:4f:35:
         ab:46:a0:4a:0f:ae:b2:03:08:e2:9e:5e:65:89:2e:d0:3d:f6:
         79:9d:3c:97:cd:ec:2c:d2:5c:04:22:8c:3b:80:78:eb:5a:c2:
         80:b3:55:70:ae:d8:c3:af:be:c8:61:8f:12:0a:ff:a3:4d:2c:
         d5:26:0e:62:e8:46:c4:82:a2:74:b4:09:d0:59:0e:15:a2:39:
         95:fb:98:1d:36:33:3a:56:37:ed:4c:bb:82:73:f7:72:48:22:
         2f:29:89:ff:84:9a:5e:6e:3a:b7:a6:1f:da:7a:8c:e9:9e:a2:
         f8:05:9e:b2:8f:05:ed:3a:b5:fd:ac:46:1f:3a:72:44:a8:f5:
         67:b7:bb:48
-----BEGIN X509 CRL-----
MIIBjDB2AgEBMA0GCSqGSIb3DQEBCwUAMA0xCzAJBgNVBAMMAmNhFw0yMDA3MTQw
MzQyNTlaFw0yMTA3MTQwMzQyNTlaMDUwMwIUBuGnMmSFgnx9Q/MRygPR+Wh3P80X
DTIwMDcxNDAzNDI1OVowDDAKBgNVHRUEAwoBBDANBgkqhkiG9w0BAQsFAAOCAQEA
s0ALaVSigEpFECj+JonpVRNPwVxv1IhCSbqFQVNkiiXSbwMUIVY52e9fuSOt3ZCk
PoQyvPJ22/D66xZonolib6xQhOStsz3+pm0KLs+r6SrfRjClOH4UOHZmbhfd7Pj8
r2aCLtwGQ88XZnos2GvUmBkGgW3dXXdGeQD0xk81q0agSg+usgMI4p5eZYku0D32
eZ08l83sLNJcBCKMO4B461rCgLNVcK7Yw6++yGGPEgr/o00s1SYOYuhGxIKidLQJ
0FkOFaI5lfuYHTYzOlY37Uy7gnP3ckgiLymJ/4SaXm46t6Yf2nqM6Z6i+AWeso8F
7Tq1/axGHzpyRKj1Z7e7SA==
-----END X509 CRL-----

# Merge ca.cer and ca.crl into a single file
cat ca.cer ca.crl > ca

# Verify valid certificate
openssl verify -crl_check -CAfile ca valid.cer
valid.cer: OK

# Verify revoked certificate
# The status is revoked
openssl verify -crl_check -CAfile ca revoked.cer 
CN = revoked
error 23 at 0 depth lookup: certificate revoked
error revoked.cer: verification failed
```

### OCSP (Online Certificate Status Protocol)
```
# Start OCSP responder
openssl ocsp -index index -rmd sha256 -CA ca.cer -rsigner ca.cer -rkey ca.key -port 8444

# Request revocation status for revoked certificate. The status is revoked.
# It also stores OCSP resquest and response to local file req.out and resp.out respectively.
openssl ocsp -sha256 -issuer ca.cer -cert revoked.cer -text -reqout req.out -respout resp.out -url http://localhost:8444
OCSP Request Data:
    Version: 1 (0x0)
    Requestor List:
        Certificate ID:
          Hash Algorithm: sha256
          Issuer Name Hash: 6D55A0A38E5290F86E181E6A9C8E91F3AA5EE0E0CA89D5B77FFF149591933271
          Issuer Key Hash: B8E5F40879373A7FCC8E0598BB7543DD15F723259E4CCF9DD410BB42997A70BC
          Serial Number: 06E1A7326485827C7D43F311CA03D1F968773FCD
    Request Extensions:
        OCSP Nonce: 
            0410CB134DF748A2E573211D7B3B9BC0CD17
OCSP Response Data:
    OCSP Response Status: successful (0x0)
    Response Type: Basic OCSP Response
    Version: 1 (0x0)
    Responder Id: CN = ca
    Produced At: Jul 18 14:09:51 2020 GMT
    Responses:
    Certificate ID:
      Hash Algorithm: sha256
      Issuer Name Hash: 6D55A0A38E5290F86E181E6A9C8E91F3AA5EE0E0CA89D5B77FFF149591933271
      Issuer Key Hash: B8E5F40879373A7FCC8E0598BB7543DD15F723259E4CCF9DD410BB42997A70BC
      Serial Number: 06E1A7326485827C7D43F311CA03D1F968773FCD
    Cert Status: revoked
    Revocation Time: Jul 14 03:42:59 2020 GMT
    Revocation Reason: superseded (0x4)
    This Update: Jul 18 14:09:51 2020 GMT

    Response Extensions:
        OCSP Nonce: 
            0410CB134DF748A2E573211D7B3B9BC0CD17
    Signature Algorithm: sha256WithRSAEncryption
         95:8f:2a:08:06:10:2f:13:b8:29:c5:15:c5:e6:16:10:34:2c:
         fc:6b:6f:69:19:ff:9f:15:45:4f:33:80:a6:d7:3d:8d:5f:b3:
         de:fe:4f:5e:47:c3:ef:25:22:5b:9f:0b:c3:67:1c:01:f8:86:
         f2:53:14:38:2e:4f:c4:11:2c:9f:c1:8a:49:4f:18:a7:76:99:
         ac:5d:f6:67:1d:b5:0d:62:0f:79:dd:50:52:b0:f6:c9:50:82:
         84:52:65:ab:c8:d4:18:3d:4c:41:65:4b:6e:29:bf:77:ef:84:
         af:9a:78:e6:a0:e9:aa:df:00:e2:8a:3d:9f:02:22:21:01:62:
         3d:87:13:37:46:b4:a9:f2:60:d7:8a:1a:fa:b7:be:64:df:41:
         49:96:95:d8:ce:1f:5e:69:77:4c:c1:6f:59:22:45:e8:b6:7b:
         29:eb:a3:63:b4:11:d8:a3:d1:66:46:d6:f7:fd:47:d3:74:12:
         8d:e1:21:36:f8:b4:e4:ec:75:17:dc:f5:00:65:e9:dc:80:8b:
         97:51:24:65:e8:0d:1e:05:96:02:cd:c5:fe:e7:0e:43:56:a6:
         a3:c6:fe:ea:84:2a:0b:98:84:eb:c4:7d:0e:cf:95:66:7d:06:
         23:a5:16:0d:87:4f:09:90:7e:22:bb:93:02:00:02:bf:91:b7:
         5f:84:6f:70
Certificate:
...
Response verify OK
revoked.cer: revoked
	This Update: Jul 19 08:53:50 2020 GMT
	Reason: superseded
	Revocation Time: Jul 14 03:42:59 2020 GMT

# Re-use the stored OCSP request and response,
# then it's unnecessary to speicfy the certificate and connect to the OCSP responder.
openssl ocsp -issuer ca.cer -text -reqin req.out -respin resp.out
OCSP Request Data:
    Version: 1 (0x0)
    Requestor List:
        Certificate ID:
          Hash Algorithm: sha256
          Issuer Name Hash: 6D55A0A38E5290F86E181E6A9C8E91F3AA5EE0E0CA89D5B77FFF149591933271
          Issuer Key Hash: B8E5F40879373A7FCC8E0598BB7543DD15F723259E4CCF9DD410BB42997A70BC
          Serial Number: 06E1A7326485827C7D43F311CA03D1F968773FCD
    Request Extensions:
        OCSP Nonce: 
            0410CB134DF748A2E573211D7B3B9BC0CD17
OCSP Response Data:
    OCSP Response Status: successful (0x0)
    Response Type: Basic OCSP Response
    Version: 1 (0x0)
    Responder Id: CN = ca
    Produced At: Jul 18 14:09:51 2020 GMT
    Responses:
    Certificate ID:
      Hash Algorithm: sha256
      Issuer Name Hash: 6D55A0A38E5290F86E181E6A9C8E91F3AA5EE0E0CA89D5B77FFF149591933271
      Issuer Key Hash: B8E5F40879373A7FCC8E0598BB7543DD15F723259E4CCF9DD410BB42997A70BC
      Serial Number: 06E1A7326485827C7D43F311CA03D1F968773FCD
    Cert Status: revoked
    Revocation Time: Jul 14 03:42:59 2020 GMT
    Revocation Reason: superseded (0x4)
    This Update: Jul 18 14:09:51 2020 GMT

    Response Extensions:
        OCSP Nonce: 
            0410CB134DF748A2E573211D7B3B9BC0CD17
    Signature Algorithm: sha256WithRSAEncryption
         95:8f:2a:08:06:10:2f:13:b8:29:c5:15:c5:e6:16:10:34:2c:
         fc:6b:6f:69:19:ff:9f:15:45:4f:33:80:a6:d7:3d:8d:5f:b3:
         de:fe:4f:5e:47:c3:ef:25:22:5b:9f:0b:c3:67:1c:01:f8:86:
         f2:53:14:38:2e:4f:c4:11:2c:9f:c1:8a:49:4f:18:a7:76:99:
         ac:5d:f6:67:1d:b5:0d:62:0f:79:dd:50:52:b0:f6:c9:50:82:
         84:52:65:ab:c8:d4:18:3d:4c:41:65:4b:6e:29:bf:77:ef:84:
         af:9a:78:e6:a0:e9:aa:df:00:e2:8a:3d:9f:02:22:21:01:62:
         3d:87:13:37:46:b4:a9:f2:60:d7:8a:1a:fa:b7:be:64:df:41:
         49:96:95:d8:ce:1f:5e:69:77:4c:c1:6f:59:22:45:e8:b6:7b:
         29:eb:a3:63:b4:11:d8:a3:d1:66:46:d6:f7:fd:47:d3:74:12:
         8d:e1:21:36:f8:b4:e4:ec:75:17:dc:f5:00:65:e9:dc:80:8b:
         97:51:24:65:e8:0d:1e:05:96:02:cd:c5:fe:e7:0e:43:56:a6:
         a3:c6:fe:ea:84:2a:0b:98:84:eb:c4:7d:0e:cf:95:66:7d:06:
         23:a5:16:0d:87:4f:09:90:7e:22:bb:93:02:00:02:bf:91:b7:
         5f:84:6f:70
Certificate:
...
Response verify OK

# Request revocation status for valid certificate
# The status is unkown due to the certificate is not in the database (exactly, index file)
openssl ocsp -sha256 -issuer ca.cer -cert valid.cer -url http://localhost:8444
Response verify OK
valid.cer: unknown
	This Update: Jul 14 12:09:42 2020 GMT

# Add valid.cer to index by manual
echo -e "V\t300714034259Z\t\t`openssl x509 -noout -serial -in valid.cer | sed 's/.*=//'`\tunknown\t/CN=valid" >> index

# The updated index file
cat index
R	200813034252Z	200714034259Z,superseded	06E1A7326485827C7D43F311CA03D1F968773FCD	unknown	/CN=revoked
V	300714034259Z		06E1A7326485827C7D43F311CA03D1F968773FCC	unknown	/CN=valid

# Re-request revocation status for valid certificate
# The status is good now
openssl ocsp -sha256 -issuer ca.cer -cert valid.cer -url http://localhost:8444
Response verify OK
valid.cer: good
	This Update: Jul 14 12:14:19 2020 GMT
```

