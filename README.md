# Ansible Role: Cloudflare SSL

An Ansible role that configures SSL/TLS certificates for Cloudflare Authenticated Origin Pulls. This role supports both automatic certificate generation via CloudFlare API and manual certificate deployment.

## Features

- **Two deployment modes:**
  - 🤖 Automatic certificate generation via CloudFlare API
  - 📋 Manual certificate deployment from AWS Secrets Manager
- Installs OpenSSL
- Sets up standard SSL directories (`/etc/ssl/cloudflare`, `/etc/ssl/private`, `/etc/ssl/certs`)
- Deploys CloudFlare Origin Certificates (certificate and private key)
- Deploys the Cloudflare Authenticated Origin Pull CA certificate
- Generates DH parameters (2048-bit)
- Triggers Nginx restart upon certificate or configuration changes

## Requirements

- **Ansible:** 2.9 or higher
- **Collections:** `amazon.aws`, `community.aws`, `community.crypto`
- **Target OS:** Amazon Linux 2023
- **Web Server:** Nginx (expected handler: `restart nginx`)

## Two Deployment Modes

### Mode 1: Automatic Generation via API (Recommended for Automation)

Automatically generates certificates using the CloudFlare API during playbook execution.

**Advantages:**
- ✅ Fully automated
- ✅ No manual steps
- ✅ Perfect for CI/CD
- ✅ Easy certificate renewal

**Example:**

```yaml
- hosts: webservers
  become: yes
  roles:
    - role: cloudflare-ssl
      vars:
        ssl_hostname: "example.com"
        ssl_generate_certificate: true
        ssl_cloudflare_api_token_secret_name: "prod/environment/cloudflare-api-token"
        ssl_certificate_hostnames:
          - "example.com"
          - "www.example.com"
        ssl_certificate_validity_days: 5475  # 15 years
```

### Mode 2: Manual Deployment (Default)

Deploys pre-generated certificates from AWS Secrets Manager.

**Advantages:**
- ✅ More control over certificate generation
- ✅ No API token needed in automation
- ✅ Certificates can be reviewed before deployment

**Example:**

```yaml
- hosts: webservers
  become: yes
  roles:
    - role: cloudflare-ssl
      vars:
        ssl_hostname: "example.com"
        ssl_cloudflare_aop_ca_content: "{{ lookup('amazon.aws.secretsmanager_secret', 'prod/environment/cloudflare-aop-ca', region='us-east-1') }}"
        ssl_cloudflare_origin_cert_content: "{{ lookup('amazon.aws.secretsmanager_secret', 'prod/environment/cloudflare-origin-cert', region='us-east-1') }}"
        ssl_cloudflare_origin_key_content: "{{ lookup('amazon.aws.secretsmanager_secret', 'prod/environment/cloudflare-origin-key', region='us-east-1') }}"
```

## Setup Instructions

### For Mode 1: API Generation

1. **Create CloudFlare API Token:**
   - Go to [CloudFlare Dashboard → My Profile → API Tokens](https://dash.cloudflare.com/profile/api-tokens)
   - Click **Create Token**
   - Use template: **Origin CA** or create custom with:
     - Permissions: `SSL and Certificates - Origin CA - Create`
   - Copy the token

2. **Store API Token in AWS Secrets Manager:**
   ```bash
   aws secretsmanager create-secret \
     --name "prod/environment/cloudflare-api-token" \
     --description "CloudFlare API Token for Origin CA" \
     --secret-string "your-api-token-here" \
     --region us-east-1
   ```

3. **Run playbook with API generation enabled**

### For Mode 2: Manual Deployment

#### Option A: Use the Generator Script (Easiest)

```bash
./scripts/generate-cloudflare-cert.sh
```

#### Option B: Manual Dashboard Generation

1. **Generate CloudFlare Origin Certificate:**
   - Log into CloudFlare Dashboard
   - Navigate to **SSL/TLS → Origin Server**
   - Click **Create Certificate**
   - Select:
     - **Let CloudFlare generate a private key and a CSR** (recommended)
     - **Key type:** RSA (2048)
     - **Certificate Validity:** 15 years
     - **Hostnames:** Add your domain (e.g., `example.com`, `*.example.com`)
   - Click **Create**
   - Copy both the certificate and private key

2. **Download CloudFlare Authenticated Origin Pull CA Certificate:**
   ```bash
   curl -o origin-pull-ca.pem https://developers.cloudflare.com/ssl/static/authenticated_origin_pull_ca.pem
   ```

3. **Store Certificates in AWS Secrets Manager:**
   ```bash
   # Use the helper script
   ./scripts/setup-cloudflare-certs.sh

   # Or manually:
   aws secretsmanager create-secret \
     --name "prod/environment/cloudflare-aop-ca" \
     --secret-string "$(cat origin-pull-ca.pem)" \
     --region us-east-1

   aws secretsmanager create-secret \
     --name "prod/environment/cloudflare-origin-cert" \
     --secret-string "$(cat origin-certificate.pem)" \
     --region us-east-1

   aws secretsmanager create-secret \
     --name "prod/environment/cloudflare-origin-key" \
     --secret-string "$(cat origin-private-key.pem)" \
     --region us-east-1
   ```

## Role Variables

Variables and their default values are defined in `defaults/main.yml`:

| Variable | Default | Description |
|----------|---------|-------------|
| `ssl_hostname` | `{{ inventory_hostname }}` | The hostname for the certificate files |
| `ssl_cloudflare_aop_ca_content` | `""` | Content of the CloudFlare Authenticated Origin Pull CA certificate |
| `ssl_cloudflare_origin_cert_content` | `""` | Content of the CloudFlare Origin Certificate |
| `ssl_cloudflare_origin_key_content` | `""` | Content of the CloudFlare Origin Certificate private key |
| `ssl_aws_region` | `us-east-1` | AWS region where secrets are stored |
| `ssl_dhparam_size` | `2048` | Size of DH parameters |

## Dependencies

None.

## Example Playbook

```yaml
- hosts: webservers
  become: yes
  roles:
    - role: cloudflare-ssl
      vars:
        ssl_hostname: "example.com"
        ssl_cloudflare_aop_ca_content: "{{ lookup('amazon.aws.secretsmanager_secret', 'prod/environment/cloudflare-aop-ca', region='us-east-1') }}"
        ssl_cloudflare_origin_cert_content: "{{ lookup('amazon.aws.secretsmanager_secret', 'prod/environment/cloudflare-origin-cert', region='us-east-1') }}"
        ssl_cloudflare_origin_key_content: "{{ lookup('amazon.aws.secretsmanager_secret', 'prod/environment/cloudflare-origin-key', region='us-east-1') }}"
```

## How It Works

### Authenticated Origin Pulls (AOP)

1. **Client → CloudFlare:** Client connects to CloudFlare using CloudFlare's public certificate
2. **CloudFlare → Origin:** CloudFlare connects to your origin server using:
   - Your CloudFlare Origin Certificate (server authentication)
   - CloudFlare's client certificate (client authentication via AOP CA)
3. **Origin validates:** Your nginx validates CloudFlare's client certificate against the AOP CA

This ensures:
- ✅ Only CloudFlare can connect to your origin (blocks direct attacks)
- ✅ Encrypted connection between CloudFlare and origin
- ✅ No certificate warnings or errors

## License

MIT

## Author Information

This role was maintained by the DevOps Team.
