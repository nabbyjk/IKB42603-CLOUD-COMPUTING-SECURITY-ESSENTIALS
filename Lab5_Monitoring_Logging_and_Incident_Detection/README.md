# IKB42603 Cloud Computing Security Essentials
## Lab 5.1 — Management Plane Audit and BCDR

---

# Objective

This lab focuses on management plane auditing, audit trail integrity, backup and disaster recovery, Recovery Point Objective (RPO), Recovery Time Objective (RTO), and the difference between object versioning and a separate backup destination.

The lab uses Kali Linux, Docker, LocalStack, and AWS CLI commands to reconstruct a management plane audit trail, seal the trail with SHA-256, perform a tamper-verification test, create a primary and disaster recovery (DR) backup bucket, simulate a destructive incident, restore the objects, measure RTO, and compare versioning with a separate backup path.

---

# Environment Setup

## Step 1 — Start LocalStack

### Command

    docker run -d --name localstack -p 4566:4566 -e SERVICES=s3,iam,sts localstack/localstack

### Result / Output

LocalStack container was started.

### Explanation

LocalStack provides a local AWS-compatible environment for performing the cloud security exercises.

### Screenshot

    01_LocalStack_AWS_CLI_Connection.png

### Move Screenshot

    mv ~/Pictures/01_LocalStack_AWS_CLI_Connection.png evidence/


## Step 2 — Wait for LocalStack Services

### Command

    until curl -sf http://localhost:4566/_localstack/health >/dev/null; do sleep 2; done

### Result / Output

The command returned to the terminal prompt after LocalStack became ready.

### Explanation

The command waits until the LocalStack service is actually available before continuing with AWS CLI operations.

### Screenshot

    01_LocalStack_AWS_CLI_Connection.png

### Move Screenshot

    mv ~/Pictures/01_LocalStack_AWS_CLI_Connection.png evidence/


## Step 3 — Set the LocalStack Endpoint

### Command

    export EP='--endpoint-url=http://localhost:4566'

### Result / Output

The LocalStack endpoint was assigned to the `EP` environment variable.

### Explanation

The variable allows the AWS CLI commands to communicate with the local LocalStack endpoint instead of the real AWS environment.

### Screenshot

    01_LocalStack_AWS_CLI_Connection.png

### Move Screenshot

    mv ~/Pictures/01_LocalStack_AWS_CLI_Connection.png evidence/


## Step 4 — Verify AWS CLI Connectivity

### Command

    aws $EP sts get-caller-identity

### Result / Output

    {
        "UserId": "000000000000",
        "Account": "000000000000",
        "Arn": "arn:aws:iam::000000000000:root"
    }

### Explanation

The output confirms that the AWS CLI can communicate with LocalStack successfully.

### Screenshot

    01_LocalStack_AWS_CLI_Connection.png

### Move Screenshot

    mv ~/Pictures/01_LocalStack_AWS_CLI_Connection.png evidence/

---

# TASK A1 — Reconstruct the Management Plane Audit Trail

The purpose of Task A1 is to reconstruct administrative management-plane activity from LocalStack's request log, filter security-relevant events, create a cryptographic digest, store the evidence separately, and verify whether tampering can be detected.

In a production cloud environment, management plane events are recorded by an audit service such as CloudTrail and stored separately so that the audit record can survive an attack against the account being audited. LocalStack CloudTrail is not included in the course licence, so the lab reconstructs the trail from the platform request log instead.

---

## Step 5 — Create the Audit Trail Bucket

### Command

    aws $EP s3api create-bucket --bucket miit-audit-trail

    aws $EP s3api put-bucket-versioning --bucket miit-audit-trail \
      --versioning-configuration Status=Enabled

### Result / Output

The `miit-audit-trail` bucket was created and versioning was enabled.

### Explanation

The audit trail requires a dedicated storage location. Versioning provides additional protection against accidental deletion or overwriting of stored audit evidence.

### Screenshot

    02_Audit_Trail_Bucket_Creation.png

### Move Screenshot

    mv ~/Pictures/02_Audit_Trail_Bucket_Creation.png evidence/


## Step 6 — Record the Baseline Log Position

### Command

    BEFORE=$(docker logs localstack 2>&1 | wc -l)
    echo "baseline: $BEFORE lines"

### Result / Output

The current number of LocalStack log lines was recorded as the baseline.

### Explanation

The baseline marks the beginning of the observation window so that only the administrative actions generated during the task are extracted.

### Screenshot

    03_Administrative_Management_Actions.png

### Move Screenshot

    mv ~/Pictures/03_Administrative_Management_Actions.png evidence/


## Step 7 — Generate Administrative Management Plane Activity

### Command

    aws $EP s3api create-bucket --bucket miit-throwaway

    aws $EP iam create-user --user-name TempContractor

    aws $EP iam attach-user-policy --user-name TempContractor \
      --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

    aws $EP s3api delete-bucket --bucket miit-throwaway

### Result / Output

The administrative actions were successfully performed:

- Created the `miit-throwaway` bucket.
- Created the `TempContractor` IAM user.
- Attached the `AdministratorAccess` policy to the user.
- Deleted the `miit-throwaway` bucket.

### Explanation

These are management plane actions because they modify the cloud environment itself rather than generating application traffic. An attacker could use similar actions to create identities, grant administrative privileges, and destroy cloud resources.

### Screenshot

    03_Administrative_Management_Actions.png

### Move Screenshot

    mv ~/Pictures/03_Administrative_Management_Actions.png evidence/


## Step 8 — Extract the Management Plane Trail

### Command

    docker logs localstack 2>&1 | tail -n +$((BEFORE+1)) \
      | grep -E 'AWS [a-z0-9-]+\.[A-Za-z]+ => ' > mgmt-trail.log

    wc -l mgmt-trail.log

### Result / Output

The LocalStack request log was extracted into `mgmt-trail.log`.

### Explanation

LocalStack records AWS API calls in its request log. The extracted file represents the reconstructed management plane audit trail for the observation window.

### Screenshot

    04_Management_Plane_Audit_Trail.png

### Move Screenshot

    mv ~/Pictures/04_Management_Plane_Audit_Trail.png evidence/


## Step 9 — Filter Security-Relevant Management Events

### Command

    grep -E '\.(CreateUser|AttachUserPolicy|DeleteBucket|PutBucketPolicy|DeleteUser|CreateAccessKey|DeleteAccessKey|PutUserPolicy|DeleteUserPolicy)' mgmt-trail.log

### Result / Output

The management trail was filtered to identify administrative operations that can change identity, permissions, or cloud resources.

### Explanation

A raw management trail can contain many events. Filtering allows investigators to focus on security-relevant actions, especially actions involving identity, privilege, and resource deletion.

### Screenshot

    05_Filtered_Management_Plane_Events.png

### Move Screenshot

    mv ~/Pictures/05_Filtered_Management_Plane_Events.png evidence/


## Step 10 — Create a SHA-256 Digest

### Command

    sha256sum mgmt-trail.log | tee mgmt-trail.sha256

### Result / Output

A SHA-256 digest was generated for `mgmt-trail.log`.

### Explanation

The digest acts as an integrity seal. If the contents of the audit trail are modified later, recalculating the SHA-256 hash will produce a different value.

### Screenshot

    06_Audit_Trail_SHA256_Digest.png

### Move Screenshot

    mv ~/Pictures/06_Audit_Trail_SHA256_Digest.png evidence/


## Step 11 — Store the Trail and Digest in the Audit Bucket

### Command

    aws $EP s3 cp mgmt-trail.log s3://miit-audit-trail/
    aws $EP s3 cp mgmt-trail.sha256 s3://miit-audit-trail/

### Result / Output

The management trail and SHA-256 digest were uploaded to `miit-audit-trail`.

### Explanation

The audit evidence is stored in a dedicated audit bucket rather than together with the workload data. In a real deployment, the audit store should be located in a separate account or trust boundary.

### Screenshot

    07_Audit_Trail_Stored_in_Separate_Bucket.png

### Move Screenshot

    mv ~/Pictures/07_Audit_Trail_Stored_in_Separate_Bucket.png evidence/


## Step 12 — Tamper with the Audit Trail and Verify Integrity

### Command

    sed -i '/AttachUserPolicy/d' mgmt-trail.log

    sha256sum mgmt-trail.log > check.sha256

    diff -q check.sha256 mgmt-trail.sha256

### Result / Output

The verification failed because the `AttachUserPolicy` event was removed from the local audit trail.

### Explanation

Removing an event changes the file contents and therefore changes its SHA-256 digest. The mismatch demonstrates that cryptographic hashing can detect modification of the audit record.

### Screenshot

    08_Audit_Trail_Tampering_Detected.png

### Move Screenshot

    mv ~/Pictures/08_Audit_Trail_Tampering_Detected.png evidence/


### Management Plane Investigation Notes

The reconstructed trail does not contain all the fields that a real CloudTrail record would contain.

Four important fields present in a real CloudTrail record but not fully reconstructed here are:

1. `userIdentity` — identifies the identity responsible for the API call.
2. `eventTime` — establishes when the event occurred.
3. `sourceIPAddress` — identifies the source network address of the request.
4. `requestParameters` — records important parameters supplied to the API operation.

Other useful CloudTrail fields include `userAgent`, `eventID`, `errorCode`, `readOnly`, and `managementEvent`.

The `sourceIPAddress` field is particularly useful because if the same address appears in another investigation, such as the brute-force login investigation from Lab 5, it can correlate the two investigations into one potentially related incident.

The reconstructed trail is also written by the same platform that served the API calls. An attacker with administrator privileges could potentially modify, delete, or disable the platform's logs. A real deployment prevents this by sending audit records to a separate account or trust boundary with restricted access and appropriate immutability/protection controls.

---

# TASK A2 — Backup, and Why Versioning Is Not One

The purpose of Task A2 is to create a primary storage bucket and a genuinely separate backup destination, populate the primary bucket with 200 objects, and synchronise those objects to the DR bucket.

---

## Step 13 — Create Primary and DR Backup Buckets

### Command

    aws $EP s3api create-bucket --bucket miit-primary
    aws $EP s3api create-bucket --bucket miit-dr-backup

    aws $EP s3api put-bucket-versioning --bucket miit-primary \
      --versioning-configuration Status=Enabled

    aws $EP s3api put-bucket-versioning --bucket miit-dr-backup \
      --versioning-configuration Status=Enabled

### Result / Output

The following buckets were created:

- `miit-primary`
- `miit-dr-backup`

Versioning was enabled on both buckets.

### Explanation

The primary bucket stores the working dataset while the DR bucket provides a separate recovery destination. Although both buckets use versioning, they remain separate storage locations.

### Screenshot

    09_Primary_and_DR_Backup_Buckets.png

### Move Screenshot

    mv ~/Pictures/09_Primary_and_DR_Backup_Buckets.png evidence/


## Step 14 — Generate and Upload 200 Records

### Command

    for i in $(seq 1 200); do
      echo "patient record $i - $(date)" > /tmp/rec$i.txt
    done

    aws $EP s3 sync /tmp/ s3://miit-primary/records/ \
      --exclude '*' --include 'rec*.txt'

    aws $EP s3 ls s3://miit-primary/records/ | wc -l

### Result / Output

    200

### Explanation

A dataset of 200 record files was generated and uploaded to the primary bucket. The count confirms that all 200 objects were present in the primary bucket.

### Screenshot

    10_Primary_Bucket_200_Objects.png

### Move Screenshot

    mv ~/Pictures/10_Primary_Bucket_200_Objects.png evidence/


## Step 15 — Synchronise the Primary Bucket to the DR Backup

### Command

    aws $EP s3 sync s3://miit-primary s3://miit-dr-backup

    aws $EP s3 ls s3://miit-dr-backup/records/ | wc -l

### Result / Output

    200

### Explanation

The 200 objects were successfully copied from the primary bucket to the separate DR backup bucket.

The separate backup destination provides recovery capability even when the primary storage is affected. Versioning alone is not equivalent to a separate backup because the versions remain within the same bucket and trust boundary.

### Screenshot

    11_DR_Backup_200_Objects.png

### Move Screenshot

    mv ~/Pictures/11_DR_Backup_200_Objects.png evidence/

---

# TASK A3 — Destructive Incident and Timed Restore

The purpose of Task A3 is to simulate a destructive incident against the primary bucket, verify the loss of the current objects, restore the objects from the separate DR backup, and measure the recovery time.

---

## Step 16 — Simulate the Destructive Incident

### Command

    aws $EP s3 rm s3://miit-primary/records/ --recursive

    aws $EP s3 ls s3://miit-primary/records/ | wc -l

### Result / Output

    0

### Explanation

The records in the primary bucket were deleted to simulate a destructive incident.

Because versioning was enabled, delete markers were created instead of permanently removing the previous object versions. However, the current object listing became empty.

### Screenshot

    12_Primary_Bucket_Destructive_Incident.png

### Move Screenshot

    mv ~/Pictures/12_Primary_Bucket_Destructive_Incident.png evidence/


## Step 17 — Restore the Records from the DR Backup and Measure RTO

### Command

    START=$(date +%s)

    aws $EP s3 sync s3://miit-dr-backup/records/ s3://miit-primary/records/

    END=$(date +%s)

    COUNT=$(aws $EP s3 ls s3://miit-primary/records/ | wc -l)

    echo "Objects restored: $COUNT"
    echo "MEASURED RTO (seconds): $((END-START))"

### Result / Output

    Objects restored: 200
    MEASURED RTO (seconds): 2

### Explanation

The 200 records were restored successfully from the separate DR backup. The measured Recovery Time Objective (RTO) for this lab restore was **2 seconds**.

This RTO represents the measured recovery time for 200 objects in the LocalStack lab environment. It should not be treated as a production-scale RTO.

### Screenshot

    13_Timed_Restore_Measured_RTO.png

### Move Screenshot

    mv ~/Pictures/13_Timed_Restore_Measured_RTO.png evidence/


## Step 18 — Verify the Restored Primary Bucket

### Command

    aws $EP s3 ls s3://miit-primary/records/ | wc -l

### Result / Output

    200

### Explanation

The primary bucket contained all 200 records after the restore, confirming that the recovery was successful.

### Screenshot

    15_Primary_Bucket_Post_Restore_Verification.png

### Move Screenshot

    mv ~/Pictures/15_Primary_Bucket_Post_Restore_Verification.png evidence/


---

# TASK A4 — Versioning, Delete Markers, and Recovery Paths

The purpose of Task A4 is to examine the object versions and delete markers created by the destructive incident and compare in-place versioning with recovery from a separate backup bucket.

---

## Step 19 — Count Delete Markers

### Command

    aws $EP s3api list-object-versions --bucket miit-primary \
      --prefix records/ --query 'length(DeleteMarkers)'

### Result / Output

    200

### Explanation

There were 200 delete markers because the 200 objects were deleted while versioning was enabled.

### Screenshot

    14_Versioning_Delete_Markers_and_Versions.png

### Move Screenshot

    mv ~/Pictures/14_Versioning_Delete_Markers_and_Versions.png evidence/


## Step 20 — Count Object Versions

### Command

    aws $EP s3api list-object-versions --bucket miit-primary \
      --prefix records/ --query 'length(Versions)'

### Result / Output

    400

### Explanation

The primary bucket contained 400 object versions. The versions demonstrate that versioning retained previous object states even after the delete operation.

### Screenshot

    14_Versioning_Delete_Markers_and_Versions.png

### Move Screenshot

    mv ~/Pictures/14_Versioning_Delete_Markers_and_Versions.png evidence/


## Step 21 — Verify Recovery from the Primary Bucket

### Command

    aws $EP s3 ls s3://miit-primary/records/ | wc -l

### Result / Output

    200

### Explanation

The primary bucket contained 200 objects after the DR restore.

### Screenshot

    15_Primary_Bucket_Post_Restore_Verification.png

### Move Screenshot

    mv ~/Pictures/15_Primary_Bucket_Post_Restore_Verification.png evidence/

---

# Evidence Summary

The following evidence was collected for the lab:

1. `01_LocalStack_AWS_CLI_Connection.png`
2. `02_Audit_Trail_Bucket_Creation.png`
3. `03_Administrative_Management_Actions.png`
4. `04_Management_Plane_Audit_Trail.png`
5. `05_Filtered_Management_Plane_Events.png`
6. `06_Audit_Trail_SHA256_Digest.png`
7. `07_Audit_Trail_Stored_in_Separate_Bucket.png`
8. `08_Audit_Trail_Tampering_Detected.png`
9. `09_Primary_and_DR_Backup_Buckets.png`
10. `10_Primary_Bucket_200_Objects.png`
11. `11_DR_Backup_200_Objects.png`
12. `12_Primary_Bucket_Destructive_Incident.png`
13. `13_Timed_Restore_Measured_RTO.png`
14. `14_Versioning_Delete_Markers_and_Versions.png`
15. `15_Primary_Bucket_Post_Restore_Verification.png`

---

# RTO and RPO Table

| Metric | Result | Explanation |
|---|---:|---|
| Objects in primary before incident | 200 | Initial dataset size |
| Objects in DR backup | 200 | Backup successfully synchronised |
| Objects after destructive incident | 0 | Current primary listing after deletion |
| Objects restored | 200 | All records restored from DR |
| Measured RTO | 2 seconds | Measured restore time for 200 objects in LocalStack |
| Extrapolated RTO for 1,000,000 objects | 10,000 seconds ≈ 166.67 minutes ≈ 2.78 hours | Simple linear extrapolation from the 200-object measurement |
| RPO | Time between the last pre-incident backup sync and the incident | Exact elapsed time was not timestamped during this run, so an exact numerical RPO cannot be honestly reported |

### RTO Calculation

Using the lab's measured RTO:

    2 seconds × (1,000,000 / 200)
    = 10,000 seconds
    ≈ 166.67 minutes
    ≈ 2.78 hours

This is only a simple linear estimate. Real production performance may differ because of object count, object size, network throughput, API limits, concurrency, storage performance, and other operational factors.

### RPO Explanation

RPO represents the amount of data that could potentially be lost, measured as the time between the most recent successful backup and the incident.

In this lab, the exact timestamps for the pre-incident sync and incident were not recorded, so the exact RPO duration cannot be calculated without inventing a value.

More frequent backups reduce the potential RPO, while faster restoration improves RTO.

---

# Recovery Path Comparison

| Recovery Factor | Versioning (In-Place) | Separate Backup Bucket |
|---|---|---|
| Recovery speed | Usually faster for object-level recovery because previous versions remain in the same bucket | Requires copying/restoring data from another bucket, so it may take longer |
| Survives bucket deletion? | No. Versions are stored in the same bucket | Yes, if the separate backup bucket remains unaffected |
| Survives a compromised admin credential? | Not necessarily. An administrator with access to the bucket may also affect versions | Potentially yes if the backup uses a separate trust boundary and restricted credentials |
| Survives cryptographic erasure of the KMS key? | No if the retained versions depend on the erased key | Only if the backup uses an independently protected key that remains available |
| Cost profile | Additional storage is required for retained object versions | Additional storage plus backup/restore operations are required |

---

# Short-Answer Questions

## Question 1

### Question

Name three administrative actions that would appear in a management plane trail but produce no application log at all. For each, state what an attacker gains by performing it.

### Answer

Three examples are:

1. **CreateUser**
   - Creates a new identity controlled by the attacker.
   - The attacker gains an additional account that can potentially be used for persistence or further actions.

2. **AttachUserPolicy**
   - Allows an attacker to attach a policy such as `AdministratorAccess`.
   - The attacker gains elevated privileges and can perform highly privileged cloud management actions.

3. **DeleteBucket**
   - Deletes a cloud storage resource.
   - The attacker can destroy or disrupt stored data and cause service availability or data-loss impact.

These actions can happen entirely at the management plane and may not generate application traffic because the workload itself does not receive the request.

---

## Question 2

### Question

CloudTrail log file validation, the digest sealed in Task A1, and the hash chain built in Lab 5 Task 4 all solve the same problem by the same mechanism. Explain the mechanism, and state why the digest must be written to a different trust boundary from the account it audits.

### Answer

All three mechanisms use **cryptographic hashing** to detect changes to data.

A hash is calculated from the original log content. If the log is modified, even slightly, the resulting hash changes. Comparing the stored trusted hash with a newly calculated hash can therefore reveal whether the log was modified.

The digest must be stored in a **different trust boundary** because an attacker who compromises the audited account may also gain enough privileges to modify both the log and its digest. If both are stored in the same compromised environment, the attacker could replace the digest after modifying the log and make the evidence appear valid.

A separate account or trust boundary makes it much harder for an attacker in the audited account to modify the audit evidence and its integrity record at the same time.

---

## Question 3

### Question

Distinguish RTO from RPO using your own measured figures. Which of the two is improved by taking backups more frequently, and which by restoring faster?

### Answer

**RTO (Recovery Time Objective)** is the target or measured time required to restore the service or data after an incident.

In this lab, the measured RTO was:

    2 seconds for 200 objects

**RPO (Recovery Point Objective)** is the amount of data or time that may be lost between the most recent successful backup and the incident.

Taking backups more frequently improves **RPO** because less time exists between backup points.

Restoring faster improves **RTO** because the system returns to operation more quickly.

Therefore:

- More frequent backups → improves RPO.
- Faster restoration → improves RTO.

---

## Question 4

### Question

Your measured RTO was a few seconds. Explain why you should not report that number to a board, and what you would report instead.

### Answer

The measured 2-second RTO was obtained using only 200 objects in a LocalStack laboratory environment. It does not represent the performance of a real production environment containing potentially millions of objects.

Instead of reporting the 2-second value as a production guarantee, I would report a validated production RTO based on realistic workload size, object sizes, network conditions, restore mechanisms, and testing.

For this lab, a simple linear extrapolation gives:

    2 seconds × (1,000,000 / 200)
    = 10,000 seconds
    ≈ 2.78 hours

This estimate should be clearly labelled as an extrapolation rather than a guaranteed production RTO.

---

## Question 5

### Question

Using your A4 table, state one incident that versioning survives and the separate backup does not, and one that the backup survives and versioning does not.

### Answer

An incident that **versioning can survive** is an accidental deletion or overwrite of individual objects. The previous versions remain available in the same bucket, allowing the object to be recovered.

An incident that **the separate backup can survive but versioning cannot** is deletion of the entire primary bucket or compromise of the primary storage account. Versioning is located inside the same bucket and trust boundary, while a properly separated backup can remain available in another bucket, account, or trust boundary.

---

# Conclusion

This lab demonstrated that management plane activity is an important source of security telemetry because administrative cloud actions may not appear in application logs. A management trail was reconstructed from LocalStack request logs, filtered for security-relevant events, and protected using a SHA-256 digest. The tampering test showed that modifying the trail changed its digest and caused verification to fail.

The BCDR exercise demonstrated the difference between versioning and a separate backup destination. The primary bucket contained 200 objects, which were synchronised to the DR backup. After the primary records were deleted, all 200 records were successfully restored from the DR bucket with a measured RTO of 2 seconds in the lab environment. The primary bucket also showed 200 delete markers and 400 retained versions.

The results show that versioning is useful for object-level recovery, while a separate backup provides stronger resilience against failures or compromise affecting the original bucket or trust boundary. RTO and RPO are different recovery objectives: faster restoration improves RTO, while more frequent backups reduce RPO.

---

# Cleanup

The following commands can be used to remove the lab resources after completing the assessment:

    aws $EP s3 rb s3://miit-dr-backup --force

    aws $EP s3 rb s3://miit-audit-trail --force

    aws $EP iam detach-user-policy \
      --user-name TempContractor \
      --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

    aws $EP iam delete-user --user-name TempContractor

    docker rm -f localstack

    rm -f /tmp/rec*.txt mgmt-trail.log mgmt-trail.sha256 check.sha256

---

# References

1. IKB42603 Cloud Computing Security Essentials Lab Manual — Lab 5.1: Management Plane Audit and BCDR.
2. IKB42603 Cloud Computing Security Essentials — Week 2 lecture materials.
3. IKB42603 Cloud Computing Security Essentials — Week 6 lecture materials.
4. AWS CloudTrail log file validation documentation.
5. Cloud Security Alliance (CSA) Security Guidance v5 — Domains 6 and 11.
6. IKB42603 Lab 5 — Monitoring, Logging and Incident Detection.
7. IKB42603 Lab 6 — Versioning and Delete Markers.
