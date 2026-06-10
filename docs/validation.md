# validation and troubleshooting

## reported results

The owner reports successful uploads, downloads, and certificate renewal in the completed June 2026 project. This publication contains no original logs, screenshots, or exact test dates. The checks below are instructions to repeat, not newly executed infrastructure tests.

## repeatable checks

| check | procedure | expected result |
| --- | --- | --- |
| DNS | Resolve your DuckDNS hostname | Its address matches EC2's public address |
| HTTPS | Open the hostname in a browser | Valid hostname and certificate chain; no browser warning |
| upload | Sign in as a normal user and upload a harmless sample text file | FileCloud lists it and can preview or retrieve it |
| download | Download that file and compare SHA-256 hashes with the original | Matching content hashes |
| storage | Review FileCloud storage checks and read-only S3 object metadata | Storage checks pass and the expected upload is backed by S3 |
| renewal | Run `sudo certbot renew --dry-run` | Successful simulated renewal |
| scheduler | Inspect the renewal timer and recent service logs | Active schedule and no failed renewal runs |
| access | Try accessing a private file while signed out | File content is not disclosed |

On Ubuntu use `sha256sum ORIGINAL_FILE DOWNLOADED_FILE` to compare the sample. A dry run uses a test issuance flow; it does not prove a production certificate has already rotated. After an actual renewal, check the certificate served by Apache and its new expiry date.

## troubleshooting

- **S3 access denied:** Check the attached instance role, the bucket name and region, and explicit denies in bucket policies or organization controls. Check KMS permissions if you selected SSE-KMS. Do not grant blanket administrator access to fix a storage error.
- **Hostname points elsewhere:** Check the DuckDNS job response and the EC2 public IP, especially after a stop/start. Ensure the updater is not detecting a NAT gateway address.
- **Certificate validation fails:** Check public DNS, inbound port 80, Apache routing for the ACME challenge, and any stale IPv6 record.
- **Old certificate still served:** Inspect the TLS virtual host paths and renewal deploy/reload behavior. Confirm the correct virtual host answers for the hostname.
- **Uploads fail:** Inspect FileCloud's storage check, application upload limits, available local disk space, and relevant service logs. Redact sensitive data before sharing logs.
