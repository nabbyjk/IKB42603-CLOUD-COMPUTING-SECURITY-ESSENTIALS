# Lab 3: Encryption and Key Management
# Lab 3: Encryption & Key Management

## 1. Lab Overview

This lab demonstrates encryption and key management in a cloud-native environment using OpenSSL, Docker, Nginx, AWS CLI, and LocalStack.

The lab focuses on three security areas:

- Data integrity and authenticity
- Encryption in transit
- Encryption at rest and key management

The main activities include RSA digital signatures, SHA-256 hashing, TLS configuration, KMS encryption, data key generation, and KMS key lifecycle management.

---

## 2. Learning Outcomes

At the end of this lab, the following concepts were demonstrated:

1. Digital signatures can be used to verify data integrity and authenticity.
2. SHA-256 hashing can be used to verify file integrity.
3. TLS protects sensitive data while it is transmitted across a network.
4. KMS can be used to create and manage encryption keys.
5. Data encryption keys can be generated using KMS.
6. KMS key lifecycle operations affect the availability of encrypted data.

---

## 3. Environment

### Tools Used

- Kali Linux
- Docker
- OpenSSL
- Nginx
- AWS CLI
- LocalStack
- AWS KMS-compatible API

### LocalStack Endpoint

    http://localhost:4566

### Working Directory

    ~/IKB42603-CLOUD-COMPUTING-SECURITY-ESSENTIALS/Lab3_Encryption_and_Key_Management

---

# 4. Task 1 — Digital Signature

## 4.1 Create the Record

The record file was created using:

    echo "Patient: Ahmad, Diagnosis: confidential" > record.txt
    cat record.txt

The record contains confidential information that will be used to demonstrate digital signatures and integrity verification.

![Record File](screenshots/SS01_Task1_AES_Encrypted.png)

---

## 4.2 Generate RSA Key Pair

An RSA private key was generated using OpenSSL:

    openssl genrsa -out private.pem 2048

The corresponding public key was extracted:

    openssl rsa -in private.pem -pubout -out public.pem

The generated key files were verified:

    ls -l private.pem public.pem

The private key is used to create the digital signature, while the public key is used to verify the signature.

![RSA Key Pair](screenshots/SS02_Task1_AES_Decryption_Match.png)

---

## 4.3 Generate the RSA Digital Signature

The record was signed using the RSA private key and SHA-256:

    openssl dgst -sha256 -sign private.pem -out record.sig record.txt

The generated signature file was verified:

    ls -l record.sig

The digital signature provides integrity and authenticity protection for the record.

![RSA Signature](screenshots/SS03_Task2_RSA_KeyPair.png)

---

## 4.4 Verify the RSA Signature

The signature was verified using the public key:

    openssl dgst -sha256 -verify public.pem -signature record.sig record.txt

The result was:

    Verified OK

The `Verified OK` result confirms that the signature is valid and that the contents of the record match the data that was originally signed.

![RSA Signature Verified](screenshots/SS04_Task2_RSA_Signature_Verified.png)

---

# 5. Task 2 — File Integrity Verification

The SHA-256 hash of the record was calculated using:

    sha256sum record.txt

The resulting hash was:

    9345a32351cc1ad03e8b318059b753da6cd4e325688da97a01599b32bc945dd5  record.txt

SHA-256 provides a cryptographic fingerprint of the file. If the contents of the file are modified, its hash value will also change.

![SHA-256 Hash](screenshots/SS05_Task3_TLS_HTTPS.png)

---

# 6. Task 3 — Encryption in Transit Using TLS

## 6.1 Generate a Self-Signed Certificate

A self-signed RSA certificate and private key were generated using OpenSSL:

    openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem \
    -days 7 -nodes -subj '/CN=localhost'

The generated certificate and private key were used to configure the HTTPS server.

![TLS Certificate](screenshots/SS06_Task4_LocalStack_KMS.png)

---

## 6.2 Configure Nginx

An Nginx configuration file was created using:

    nano nginx.conf

The configuration used HTTPS on port 443:

    events {}

    http {
        server {
            listen 443 ssl;
            server_name localhost;

            ssl_certificate /etc/nginx/cert.pem;
            ssl_certificate_key /etc/nginx/key.pem;

            location / {
                root /usr/share/nginx/html;
            }

            location = /record.txt {
                alias /usr/share/nginx/html/record.txt;
            }
        }
    }

The configuration enables encrypted communication between the client and the Nginx server.

---

## 6.3 Run the Nginx HTTPS Container

The Nginx container was started using:

    docker run --rm -d --name tls -p 8443:443 \
    -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
    -v $(pwd)/cert.pem:/etc/nginx/cert.pem:ro \
    -v $(pwd)/key.pem:/etc/nginx/key.pem:ro \
    -v $(pwd)/record.txt:/usr/share/nginx/html/record.txt:ro \
    nginx

The running container was verified using:

    docker ps

The `tls` container was successfully running with port `8443` mapped to HTTPS port `443`.

![Nginx TLS Container](screenshots/SS07_Task4_KMS_KeyID.png)

---

## 6.4 Access the Record Through HTTPS

The record was accessed through the HTTPS endpoint:

    curl -k https://localhost:8443/record.txt

The result was:

    Patient: Ahmad, Diagnosis: confidential

The successful response demonstrates that the record can be accessed through an encrypted HTTPS connection.

![HTTPS Record Access](screenshots/SS08_Task4_KMS_Encryption.png)

---

# 7. Task 4 — Encryption at Rest Using AWS KMS

## 7.1 Start LocalStack

The LocalStack container was started using:

    docker start localstack

The running containers were verified:

    docker ps

The LocalStack endpoint was configured:

    export EP=http://localhost:4566

The available KMS keys were checked:

    aws $EP kms list-keys

LocalStack provides a local AWS-compatible environment for testing KMS operations.

![LocalStack KMS](screenshots/SS09_Task5_Data_Key.png)

---

## 7.2 Create the Tenant-A Master Key

A KMS master key was created using:

    aws $EP kms create-key --description 'CCSE tenant-A master key'

The original tenant-A Key ID used during the lab was:

    d2d20377-1d97-4a31-81a1-2a976384b329

The key ID was assigned to the environment variable:

    KEY_A=d2d20377-1d97-4a31-81a1-2a976384b329

The KMS master key is used to perform encryption operations and protect data encryption keys.

![Tenant A KMS Key](screenshots/SS10_Task5_Data_Key_Files.png)

---

## 7.3 Encrypt Data Using KMS

The plaintext value `hello` was encrypted using the KMS key:

    aws $EP kms encrypt \
    --key-id $KEY_A \
    --plaintext "$(echo -n 'hello' | base64)" \
    --query CiphertextBlob \
    --output text

The command returned the encrypted ciphertext blob.

This demonstrates encryption at rest using a KMS-managed key.

![KMS Encryption](screenshots/SS11_Task5_Envelope_Encryption.png)

---

## 7.4 Generate a Data Key

An AES-256 data key was generated using:

    aws $EP kms generate-data-key \
    --key-id $KEY_A \
    --key-spec AES_256 \
    --query '[Plaintext,CiphertextBlob]' \
    --output text

The plaintext data key obtained during the lab was stored as Base64:

    echo '8uovqV1tAAZ0YP7qCoSR2SXN/hEaaEXr/+GwwTcuHCY=' > datakey.b64

The generated data key can be used for symmetric encryption while its encrypted form can be protected using the KMS master key.

![KMS Data Key](screenshots/SS12_Task5_Plaintext_Key_Removed.png)

---

# 8. Task 5 — Encryption Using the Data Key

## 8.1 Decode the Data Key

The Base64-encoded data key was decoded:

    base64 -d datakey.b64 > datakey.bin

The resulting binary file contains the data encryption key.

---

## 8.2 Encrypt the Record

The record was encrypted using AES-256-CBC:

    openssl enc -aes-256-cbc \
    -in record.txt \
    -out record.enc \
    -pass file:./datakey.bin

The encrypted file was verified:

    ls -l record.enc

The record was successfully converted into encrypted ciphertext using the generated data key.

![Encrypted Record](screenshots/SS13_Task6_TenantB_KeyID.png)

---

# 9. Task 6 — KMS Key Lifecycle Management

## 9.1 Create Tenant-B Master Key

A separate KMS master key was created for tenant-B:

    aws $EP kms create-key --description 'CCSE tenant-B master key'

The tenant-B Key ID used during the lab was:

    87058c75-dd26-4b57-820f-ac483148f691

The key ID was assigned using:

    KEY_B=87058c75-dd26-4b57-820f-ac483148f691

The separate key demonstrates logical separation of encryption keys between tenants.

![Tenant B KMS Key](screenshots/SS14_Task6_Cryptographic_Erasure_Failed.png)

---

## 9.2 Disable Tenant-A Key

The tenant-A key was disabled:

    aws $EP kms disable-key --key-id $KEY_A

Disabling the key prevents normal cryptographic operations from being performed using the key.

![KMS Key Disabled](screenshots/SS15_Task7_Original_SHA256.png)

---

## 9.3 Schedule Tenant-A Key for Deletion

The key was scheduled for deletion:

    aws $EP kms schedule-key-deletion \
    --key-id $KEY_A \
    --pending-window-in-days 7

The key entered the `PendingDeletion` state with a seven-day waiting period.

This demonstrates the importance of key lifecycle management because deleting or disabling a key can affect access to encrypted data.

![KMS Key Pending Deletion](screenshots/SS16_Task7_Tampered_SHA256.png)

---

## 9.4 Attempt Decryption While Key Is Pending Deletion

The encrypted data key was decoded:

    base64 -d datakey.enc > datakey.enc.bin

A decryption attempt was performed:

    aws $EP kms decrypt \
    --ciphertext-blob fileb://datakey.enc.bin

The operation returned:

    KMSInvalidStateException

The error occurred because the KMS key was in the `PendingDeletion` state.

This demonstrates that encrypted data may become inaccessible when the encryption key required for decryption is unavailable.

![KMS Decryption Error](screenshots/SS17_Task7_Hash_Chain.png)

---

# 10. Task 7 — LocalStack Restart and KMS Key Recreation

## 10.1 Restart LocalStack

LocalStack was restarted using:

    docker start localstack

The container status was verified:

    docker ps

LocalStack successfully returned to the running state.

![LocalStack Restart](screenshots/SS18_Final_RSA_Verification.png)

---

## 10.2 Check the KMS Keys

The KMS key list was checked:

    aws --endpoint-url=http://localhost:4566 kms list-keys

The result showed:

    {
        "Keys": []
    }

This occurred because the previous LocalStack KMS state was no longer available after the environment was restarted.

---

## 10.3 Create a New Tenant-A Master Key

A new tenant-A master key was created:

    aws --endpoint-url=http://localhost:4566 kms create-key \
    --description 'CCSE tenant-A master key'

The new Key ID generated during the lab was:

    b8119053-facb-4703-9c20-ce740bca889e

The new key was verified using:

    aws --endpoint-url=http://localhost:4566 kms list-keys

The newly created key appeared in the KMS key list.


---

# 11. Task 8 — Final Verification

## 11.1 Verify SHA-256 Integrity

The SHA-256 hash of the record was calculated again:

    sha256sum record.txt

The hash remained:

    9345a32351cc1ad03e8b318059b753da6cd4e325688da97a01599b32bc945dd5  record.txt

The matching hash confirms that the record contents remained unchanged.


---

## 11.2 Verify RSA Digital Signature

The RSA signature was verified again:

    openssl dgst -sha256 -verify public.pem -signature record.sig record.txt

The result was:

    Verified OK

The successful verification confirms that the signed record remained authentic and unchanged.


---

# 12. GitHub Documentation

The project repository was opened:

    cd ~/IKB42603-CLOUD-COMPUTING-SECURITY-ESSENTIALS

The Lab 3 directory was verified:

    ls Lab3_Encryption_and_Key_Management

The Git status was checked:

    git status

The Lab 3 README was added:

    git add Lab3_Encryption_and_Key_Management/README.md

The changes were committed:

    git commit -m "Add Lab 3 encryption and key management documentation"

The changes were pushed to GitHub:

    git push

The latest commit was verified:

    git log -1 --oneline

The Lab 3 documentation was successfully committed and pushed to the GitHub repository.


---

# 13. Security Best-Practices Checklist

- [x] RSA digital signatures were used to verify data authenticity and integrity.
- [x] SHA-256 hashing was used to verify file integrity.
- [x] TLS was configured to protect data in transit.
- [x] AWS KMS was used to demonstrate encryption at rest.
- [x] Data encryption keys were generated using KMS.
- [x] Separate KMS keys were demonstrated for different tenants.
- [x] KMS key lifecycle operations were tested.
- [x] The impact of key deletion on encrypted data was demonstrated.

---

# 14. Conclusion

This lab demonstrated important encryption and key management techniques in a cloud-native environment.

RSA digital signatures were used to provide data authenticity and integrity, while SHA-256 was used to verify that the record had not been modified.

TLS was configured using Nginx to protect data in transit. AWS KMS through LocalStack was then used to demonstrate encryption at rest, data key generation, key disabling, and key deletion.

The lab also demonstrated that encryption key availability is critical because encrypted data may become inaccessible when the required KMS key is disabled or pending deletion.

---

# 15. Cleanup & Teardown

After all lab activities and documentation have been completed, the Nginx container can be stopped:

    docker stop tls

The LocalStack container can also be stopped:

    docker stop localstack

The generated lab files can be retained for documentation and assessment purposes.
