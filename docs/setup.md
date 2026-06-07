# setup guide

This guide recreates the reported architecture on a fresh deployment. It uses an Ubuntu FileCloud AWS Marketplace image, an EC2 IAM role, a private S3 bucket, Apache, and Certbot HTTP-01 validation. These are explicit reproduction choices, not claims about the exact original configuration. Record the selected AMI ID, Ubuntu version, FileCloud version, region, and instance type in your own deployment notes.

## 1. prerequisites

Have an AWS account with permission to provision EC2, S3, IAM roles, and security groups; a FileCloud license or trial; and a DuckDNS account. Choose an Ubuntu-based FileCloud image from the official vendor listing and check its supported instance sizes and license charges. Follow the vendor's [AWS deployment instructions](https://docs.filecloud.com/fcdoc/latest/server/filecloud-administrator-guide/installing-filecloud-server/installation/amazon-web-services-aws-installation/filecloud-on-aws-user-deployment-guide).

Use these placeholders consistently:

| placeholder | replace with |
| --- | --- |
| `YOUR_BUCKET_NAME` | a new globally unique bucket name |
| `YOUR_SUBDOMAIN` | your DuckDNS subdomain, without `.duckdns.org` |
| `YOUR_SUBDOMAIN.duckdns.org` | your complete hostname |
| `YOUR_ADMIN_IP/32` | your trusted administration IPv4 address |

Create the resources in one AWS region. Keep all real secrets outside this repository.

## 2. create storage and permissions

1. In S3, create a general purpose bucket in the chosen region. Keep all Block Public Access settings enabled, use Bucket owner enforced object ownership, and select SSE-S3 encryption for this example.
2. Replace `YOUR_BUCKET_NAME` in [s3-policy.json](../examples/s3-policy.json). In IAM, create a customer-managed policy using that JSON.
3. Create an IAM role with AWS service EC2 as its trusted entity. The [trust policy example](../examples/ec2-trust-policy.json) shows the corresponding trust relationship. Attach the bucket policy from the previous step to the role. It is an identity policy, not a policy to paste into the S3 bucket policy editor.
4. Attach the role as an instance profile when launching EC2, or use EC2 Actions > Security > Modify IAM role afterward.

The sample grants object read, write, delete, and multipart operations only for this bucket. It does not create buckets or administer IAM. Compare permissions with your FileCloud release and enabled features before use. SSE-KMS would also require permissions for the specific KMS key. [AWS documents how S3 actions map to bucket and object resources](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-with-s3-policy-actions.html).

## 3. launch ubuntu and initialize filecloud

1. Launch the official Ubuntu-based FileCloud AMI in a subnet with an internet gateway route and a public IPv4 address. Select a vendor-supported instance type and adequate encrypted EBS storage for the OS, database, and application.
2. Attach the instance role. Allow TCP 22 only from `YOUR_ADMIN_IP/32`. Initially restrict web access to your administration IP while completing initialization. Do not expose the database port.
3. Connect using the image's documented Ubuntu SSH account and your privately stored SSH key. Check the installed Ubuntu and FileCloud versions against the vendor requirements.
4. Complete the AMI's first-run instructions, set a unique administrator password, install the license, and run its installation checks. Configure the administrator email and disable or remove installation endpoints as directed by the vendor.
5. Confirm Apache is the serving web server with `sudo systemctl status apache2`. If the selected image uses a different server, use that server's Certbot integration instead of the Apache commands below.

## 4. configure s3 managed storage

Before uploading any files, open FileCloud's administrator storage settings, select Amazon S3 managed storage, enter the bucket and region, and enable **Use IAM role**. Leave access-key fields empty for this role-based setup. Save and run the available storage checks.

Use a new bucket. Switching an existing populated FileCloud installation to S3 does not migrate its files automatically. Manage the bucket's file contents through FileCloud. Consult [FileCloud managed S3 storage](https://docs.filecloud.com/fcdoc/latest/server/filecloud-administrator-guide/filecloud-site-setup/storage-settings/filecloud-managed-storage/setting-up-filecloud-managed-s3-storage) for release-specific fields.

## 5. configure duckdns

Create your chosen subdomain in DuckDNS and set its IPv4 address to the instance's public IPv4 address. Create a private curl configuration on the server:

```sh
sudo install -d -m 700 /etc/duckdns
sudo install -m 600 /dev/null /etc/duckdns/update.conf
sudoedit /etc/duckdns/update.conf
```

Enter the following locally. Replace the token placeholder privately; never commit the resulting file:

```text
url = "https://www.duckdns.org/update"
get
silent
show-error
fail
data-urlencode = "domains=YOUR_SUBDOMAIN"
data-urlencode = "token=ENTER_TOKEN_LOCALLY"
data-urlencode = "ip="
```

An empty `ip` lets DuckDNS detect the request's public IPv4 address. This assumes the instance connects directly through its public address, not a separate NAT gateway. Test the update:

```sh
sudo curl --config /etc/duckdns/update.conf
getent ahostsv4 YOUR_SUBDOMAIN.duckdns.org
```

Expect `OK` and the instance's public IPv4 address. Use `sudo crontab -e` to add a periodic update:

```cron
*/5 * * * * /usr/bin/curl --config /etc/duckdns/update.conf
```

The token stays in a root-only file rather than the cron command. Configure local cron mail or another monitor to catch failures, including a `KO` response. See the [DuckDNS update specification](https://www.duckdns.org/spec.jsp).

## 6. enable https and renewal

Ensure the hostname resolves correctly. Allow inbound TCP 80 for public HTTP-01 validation and TCP 443 for intended HTTPS clients. Keep SSH restricted. If an IPv6 DNS record exists, it must also reach the server correctly.

Set the Apache site's `ServerName` to your hostname using the existing FileCloud virtual host, then validate with `sudo apache2ctl configtest`. Avoid replacing the vendor's application routing configuration.

For an Ubuntu installation without an existing Certbot package or renewal mechanism:

```sh
sudo apt update
sudo apt install certbot python3-certbot-apache
sudo certbot --apache -d YOUR_SUBDOMAIN.duckdns.org --redirect
sudo systemctl enable --now certbot.timer
sudo systemctl list-timers --all | grep certbot
sudo certbot renew --dry-run
```

Follow the certificate client's prompts for your contact email and agreement. Configure FileCloud's public server URL as `https://YOUR_SUBDOMAIN.duckdns.org` and verify browser login. Inspect `sudo certbot certificates` and the Apache TLS virtual host to confirm it uses the managed certificate paths. The renewal installation must reload Apache so it serves the renewed certificate.

If Certbot is already installed through snap or another mechanism, use its existing scheduler instead of creating a duplicate. Keep port 80 reachable for future HTTP-01 challenges. The [Certbot guide](https://eff-certbot.readthedocs.io/en/stable/using.html#renewing-certificates) explains renewal testing and installers.

## 7. validate and maintain

Run the [validation checklist](validation.md). Back up the FileCloud database and configuration with the vendor-supported procedure, and test restoration together with S3 data. Monitor disk space, service health, DNS updates, and certificate expiry. Record software versions and patch regularly. Keep credentials and backups in private storage.
