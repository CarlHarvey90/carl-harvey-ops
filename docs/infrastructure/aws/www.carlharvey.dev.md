# Hosting a Static Site on AWS — S3 + CloudFront + Route 53

**Runbook:** Static site deployment  
**Status:** Live  
**Last updated:** March 2026  
**Difficulty:** Beginner–Intermediate  

---

## Overview

This runbook documents deploying a static HTML site to AWS using S3 for storage, CloudFront as a CDN with HTTPS, and Route 53 for DNS. The domain is registered at a third-party registrar (Namecheap) and delegated to Route 53 via custom nameservers.

**Use case:** Portfolio site — single HTML file, no backend, no build step required.

---

## Architecture

```
Browser
   │
   │  HTTPS (TLS 1.3)
   ▼
┌─────────────────────────────┐
│          Route 53           │
│  example.com   A (Alias)    │──────────────────────────┐
│  www.example.com A (Alias)  │                          │
└─────────────────────────────┘                          │
                                                          ▼
                                          ┌───────────────────────────┐
                                          │         CloudFront        │
                                          │   xxxxx.cloudfront.net    │
                                          │                           │
                                          │  Cert: ACM (us-east-1)   │
                                          │  HTTP → HTTPS redirect    │
                                          │  Default root: index.html │
                                          └─────────────┬─────────────┘
                                                        │
                                                   OAC (private)
                                                        │
                                          ┌─────────────▼─────────────┐
                                          │            S3             │
                                          │   your-bucket-name        │
                                          │   Block all public access │
                                          │   index.html              │
                                          └───────────────────────────┘
```

### DNS delegation chain

```
Third-party Registrar (Namecheap, GoDaddy, etc.)
   │
   │  Custom nameservers → Route 53
   ▼
Route 53 (authoritative)
   │
   ├── example.com     A Alias → CloudFront
   ├── www.example.com A Alias → CloudFront
   └── ACM validation CNAMEs (permanent — required for cert renewal)
```

---

## Prerequisites

- AWS account with appropriate IAM permissions (S3, CloudFront, ACM, Route 53)
- Domain registered at any registrar (ability to set custom nameservers required)
- Static site file ready to deploy (renamed to `index.html`)
- AWS CLI installed and configured (optional but useful for cache invalidation)

---

## Cost

| Service | Cost |
|---|---|
| Route 53 Hosted Zone | $0.50/month |
| S3 Storage + Requests | ~$0.00 (negligible for static sites) |
| CloudFront | $0.00 (within free tier for low traffic) |
| ACM Certificate | Free |
| **Total** | **~$0.50/month** |

---

## Step 1 — Identify Your Current DNS Provider

Before starting, find out who is currently authoritative for your domain:

```bash
dig yourdomain.com NS
```

The nameservers in the answer tell you who controls your DNS. Common patterns:

```
dns1.registrar-servers.com   → Namecheap
ns1.godaddy.com              → GoDaddy
ana.ns.cloudflare.com        → Cloudflare
awsdns-xx.com                → Already Route 53
```

You'll need login access to wherever DNS currently lives to add validation records in Step 3.

> **Tip:** If you get an empty answer section, query the authoritative server directly:
> ```bash
> dig yourdomain.com NS @dns1.registrar-servers.com
> ```

---

## Step 2 — Create the S3 Bucket

1. Go to **S3 → Create bucket**
2. Bucket name: your domain name (e.g. `yourdomain.com`)
3. Region: choose your nearest region
4. **Block all public access: ON** — leave this enabled. CloudFront will access the bucket privately via OAC, so the bucket never needs to be public
5. All other settings: defaults
6. Upload your HTML file renamed to `index.html`

> **Why keep the bucket private?** With Origin Access Control (OAC), CloudFront authenticates itself to S3 using a signed request. The bucket stays completely private — only CloudFront can read it. This is more secure than the older method of making buckets public.

---

## Step 3 — Request an ACM Certificate

> **Critical:** ACM certificates for CloudFront **must** be created in **us-east-1** (US East N. Virginia). This is a hard AWS requirement regardless of where your S3 bucket or other resources live.

1. Switch your AWS console region to **us-east-1**
2. Go to **ACM → Request certificate**
3. Type: Public certificate
4. Add both:
   - `yourdomain.com`
   - `www.yourdomain.com`
5. Validation method: **DNS validation**
6. Submit the request

ACM will provide two CNAME records to prove domain ownership. The records look like:

```
Name:  _abc123def456.yourdomain.com.
Value: _xyz789....acm-validations.aws.
```

Add both CNAME records to your current DNS provider (wherever you identified in Step 1).

> **Namecheap-specific:** The Host field does not include your domain suffix — enter only the subdomain portion. Namecheap appends the domain automatically. For example:
> - Correct host: `_abc123def456`
> - Wrong host: `_abc123def456.yourdomain.com`
>
> For the `www` record, the host must include `.www`:
> - Correct host: `_abc123def456.www`

### Verifying the CNAME records are live

Don't wait for ACM — verify the records yourself first. Query your DNS provider's nameserver directly to bypass local cache:

```bash
dig _abc123def456.yourdomain.com CNAME @dns1.registrar-servers.com
```

You want `ANSWER: 1` in the response. If you get `ANSWER: 0`, the record isn't saved correctly — check for typos in the host field.

> **Common mistake:** Local DNS cache can give false negatives. Always use `@your-nameserver` to query the authoritative server directly.

Wait for ACM to show **Issued** for both domains before proceeding.

---

## Step 4 — Create the CloudFront Distribution

> **Note:** AWS updated the CloudFront console wizard in early 2026. Steps below reflect the new UI.

1. Go to **CloudFront → Create distribution**
2. Distribution type: **Single website or app**
3. Distribution name: something descriptive (e.g. `yourdomain.com.distribution`)
4. Domain field: leave blank if your domain isn't in Route 53 yet
5. Click **Next**

**On the Specify origin step:**
- Origin domain: select your S3 bucket from the dropdown
- Enable **"Allow private S3 bucket access to CloudFront"** — this creates the OAC automatically

**WAF step:** Skip — not required for a static site.

6. Click through to **Review and create → Create distribution**

After creation, edit the distribution settings (**General → Settings → Edit**) to add:
- Alternate domain names: `yourdomain.com` and `www.yourdomain.com`
- Custom SSL certificate: select the ACM certificate created in Step 3
- Default root object: `index.html`

Save. Note your CloudFront domain name (e.g. `xxxxxx.cloudfront.net`) — you'll need it for DNS records.

> **Why Alias A records instead of CNAMEs?** Route 53 Alias records are AWS-specific and allow you to point an apex domain (bare `yourdomain.com`) at CloudFront. Standard DNS does not allow CNAMEs at the apex — Alias records solve this.

---

## Step 5 — Create the Route 53 Hosted Zone

1. Go to **Route 53 → Hosted zones → Create hosted zone**
2. Domain name: `yourdomain.com`
3. Type: **Public hosted zone**
4. Create

Route 53 auto-creates NS and SOA records. Copy the 4 NS record values — you'll need them in Step 7.

### Create DNS records

Click **Create record** twice:

**Record 1 — apex:**
- Name: *(blank)*
- Type: `A`
- Alias: **on**
- Route traffic to: Alias to CloudFront distribution → select your distribution
- Routing policy: Simple

**Record 2 — www:**
- Name: `www`
- Type: `A`
- Alias: **on**
- Route traffic to: same CloudFront distribution
- Routing policy: Simple

### Add the ACM validation CNAMEs to Route 53

Also add the two ACM validation CNAME records here. These must stay in Route 53 permanently — ACM uses them to auto-renew the certificate every year.

---

## Step 6 — Delegate DNS from Your Registrar to Route 53

Log into your domain registrar and change the nameservers to the 4 Route 53 NS values from Step 5.

**Namecheap:** Domain List → Manage → Nameservers → Custom DNS → enter all 4 nameservers → Save

This transfers DNS authority from your registrar to Route 53. Your registrar now only holds the domain registration — Route 53 answers all DNS queries.

---

## Step 7 — Verify

Run these in order after changing nameservers. Allow up to a few hours for propagation — most resolvers update within 30 minutes once the old TTL expires.

```bash
# 1. Confirm Route 53 nameservers are now authoritative
dig yourdomain.com NS

# 2. Confirm A records resolve to CloudFront
dig yourdomain.com A

# 3. Confirm HTTPS works and returns HTTP 200
curl -I https://yourdomain.com

# 4. Confirm www resolves and redirects correctly
curl -I https://www.yourdomain.com
```

Expected output for step 3:
```
HTTP/2 200
server: CloudFront
x-cache: Hit from cloudfront
```

---

## Updating the Site

To push a new version of the site:

1. Rename updated file to `index.html`
2. Upload to S3 bucket, replacing the existing file
3. Invalidate the CloudFront cache:

```bash
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/*"
```

> Without the invalidation, CloudFront may serve the cached old version for up to 24 hours depending on the cache TTL.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| ACM stuck on Pending validation | CNAME record not saved or has a typo | Run `dig _acme-challenge... CNAME @your-nameserver` to verify directly |
| `ANSWER: 0` on dig | Local DNS cache | Query the authoritative nameserver directly with `@ns` flag |
| CloudFront returns 403 | OAC bucket policy not applied, or wrong bucket | Check S3 bucket policy includes CloudFront OAC grant |
| CloudFront returns 403 | Default root object not set | Edit distribution → set `index.html` as default root object |
| Site shows old version after update | CloudFront cache | Run cache invalidation with `--paths "/*"` |
| www not resolving | Missing www A record in Route 53 | Add second A Alias record for `www` |
| Certificate error in browser | Cert not attached to distribution | Edit distribution settings → add ACM cert and alternate domain names |

---

## Gotchas

- **ACM must be in us-east-1** — this catches everyone the first time. CloudFront reads certificates only from this region
- **Namecheap host field** — enter only the subdomain, not the full domain. Namecheap appends the domain automatically
- **ACM validation CNAMEs are permanent** — do not delete them after validation. They are required for annual auto-renewal
- **Alias records at the apex** — you cannot use a CNAME for a bare domain in standard DNS. Route 53 Alias A records are the solution
- **Cache invalidation costs** — the first 1000 invalidation paths per month are free. `/*` counts as one path

---
