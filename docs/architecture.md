# architecture

```mermaid
flowchart TB
    U[User browser]
    D[DuckDNS DNS]
    L[Let's Encrypt ACME]
    subgraph AWS[AWS]
        subgraph EC2[Ubuntu EC2 instance]
            W[HTTPS web server]
            F[FileCloud application]
            M[(Application database and configuration)]
            C[Certbot renewal scheduler]
            J[DuckDNS update scheduler]
        end
        I[IAM identity and bucket-scoped policy]
        S[(Private S3 bucket)]
    end
    U -. DNS lookup .-> D
    D -. EC2 public IP .-> U
    U <-->|HTTPS 443| W
    W <--> F
    F <--> M
    F <-->|Authorized S3 API requests over HTTPS| S
    I -. authorizes .-> S
    I -. used by .-> F
    J -->|HTTPS public IP update| D
    C -->|ACME request and renewal| L
    L -->|HTTP-01 validation on port 80| W
    C -->|Certificate installation and reload| W
```

## request flow

DNS resolves a name; file content does not pass through DuckDNS. The browser connects to FileCloud over HTTPS. FileCloud applies user permissions before interacting with S3. IAM controls the AWS operations available to the server independently of FileCloud user permissions.

## reproduction choices

The diagram uses server-mediated file transfer and Certbot HTTP-01 validation as the reproduction design. Apache is the web server assumed by the setup guide. The original certificate challenge method and web server configuration were not supplied. Optional FileCloud direct or optimized S3 uploads are outside this guide.

An EC2 instance role is the example AWS identity. S3 holds file objects, while application metadata and configuration still need separate backups. A single EC2 instance is a single point of failure.
