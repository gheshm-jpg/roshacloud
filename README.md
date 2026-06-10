# aws filecloud project

A personal cloud file storage project built with FileCloud on an Ubuntu Amazon EC2 instance, Amazon S3, IAM access controls, DuckDNS, and Let's Encrypt HTTPS.

## cloud access

[Open the FileCloud service](https://roshafiles.duckdns.org). File access requires an authorized account.

## project overview

Completed in June 2026.

FileCloud provides the web interface and handles user authentication and file permissions. The Ubuntu EC2 instance runs the application, while an S3 bucket stores managed files. IAM policies control the application's AWS access. DuckDNS gives the server a consistent hostname, and Let's Encrypt provides the HTTPS certificate with automatic renewal.

## how it works

1. A user opens the DuckDNS hostname in a browser.
2. DNS resolves the hostname to the EC2 instance's public IP address.
3. The browser establishes an HTTPS connection to the server.
4. FileCloud authenticates the user and checks application permissions.
5. FileCloud reads or writes files in S3 using its authorized AWS identity.
6. Scheduled DNS updates track public IP changes, and certificate renewal keeps HTTPS available.

[View the architecture diagram](docs/architecture.md).

## setup guide

Follow the [setup guide](docs/setup.md) to provision EC2 and S3, configure IAM, install FileCloud, set up DuckDNS, and enable HTTPS renewal. The guide describes a reproduction path; it is not an export of the original server configuration.

## tested functionality

The project owner reports that uploads, downloads, and certificate renewal were tested successfully for the completed project. Original test logs are not included. See the [validation checklist](docs/validation.md) to repeat these checks on a new deployment.

## repository contents

| path | purpose |
| --- | --- |
| `docs/architecture.md` | component diagram and request flow |
| `docs/setup.md` | reproduction instructions |
| `docs/validation.md` | verification and troubleshooting |
| `examples/s3-policy.json` | bucket-scoped IAM policy template |
| `examples/ec2-trust-policy.json` | EC2 role trust policy template |

## security and scope

Only project documentation and placeholder configuration examples are included. AWS credentials, access keys, secret keys, API tokens, SSH keys, TLS private keys, and personal files are excluded. Supply sensitive values privately on the deployment host.

This is a personal single-instance deployment. S3 file storage does not replace backups of FileCloud's database and configuration. The documented reproduction path uses an EC2 IAM role; the original IAM authentication method and software versions were not recorded here.
