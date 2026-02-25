---
title: "MinIO on a Budget: Self-Hosted S3 for Your Apps"
date: 2025-12-28T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "self-hosted"
---

Every app eventually needs to store files somewhere. User avatars, PDF exports, CSV uploads, image attachments -- at some point you're dealing with blob storage. The default answer is "just use S3," and honestly, that's not wrong. S3 is great. But it's also another AWS bill, another set of IAM credentials to manage, and another vendor lock-in thread tying your indie app to Jeff Bezos's empire.

[MinIO](https://min.io/) gives you an S3-compatible object store that runs on your own box. Same API, same SDKs, zero cloud bills. I've been running it for two years across three projects and it's been shockingly stable for something that costs me $0/month on top of my existing VPS.

## Why S3-Compatible Matters

Here's the key insight: if your app talks to S3, it can talk to MinIO without changing a single line of code. The AWS SDK just needs a different endpoint URL. Every library, every tool, every backup script that speaks S3 -- it all works with MinIO out of the box.

That means you can develop locally against MinIO, run MinIO in production on your VPS, and if you ever outgrow it, swap in real S3 (or [Backblaze B2](https://www.backblaze.com/cloud-storage), or [Wasabi](https://wasabi.com/), or Cloudflare R2) by changing an environment variable. No vendor lock-in. No migration headaches.

## Spin It Up

Here's the [docker-compose](https://docs.docker.com/compose/) I use:

```yaml
# docker-compose.yml
version: "3"
services:
  minio:
    image: minio/minio:latest
    container_name: minio
    restart: unless-stopped
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"   # S3 API
      - "9001:9001"   # Web console
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: your-strong-password-here
    volumes:
      - minio-data:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 30s
      timeout: 5s
      retries: 3

volumes:
  minio-data:
```

```bash
docker compose up -d
```

That's it. You've got an S3-compatible object store running on port 9000 and a web console on port 9001. Hit the console in your browser, log in with the root credentials, and you'll see a clean dashboard where you can create buckets, upload files, and manage access.

**Change those default credentials.** `minioadmin/minioadmin` is the default and every bot on the internet knows it.

## The `mc` CLI: Your Best Friend

MinIO ships a CLI tool called `mc` (MinIO Client) that works like a Unix file manager for object storage. Install it:

```bash
# Linux
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && sudo mv mc /usr/local/bin/

# macOS
brew install minio/stable/mc
```

Set up an alias to your MinIO instance:

```bash
mc alias set myminio http://localhost:9000 minioadmin your-strong-password-here
```

Now you can do things like:

```bash
# Create a bucket
mc mb myminio/uploads

# Upload a file
mc cp ./photo.jpg myminio/uploads/

# List bucket contents
mc ls myminio/uploads/

# Mirror a local directory to a bucket (like rsync)
mc mirror ./assets/ myminio/uploads/assets/

# Set a bucket policy to public-read
mc anonymous set download myminio/uploads

# Get bucket disk usage
mc du myminio/uploads
```

It's fast, it's scriptable, and it works with any S3-compatible service (not just MinIO). I use it for Backblaze B2 too.

## Using MinIO in Your App

The whole point is that your app doesn't know it's talking to MinIO. It thinks it's talking to S3. Here's how that looks in practice.

### Node.js (AWS SDK v3)

```javascript
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({
  region: 'us-east-1',  // MinIO ignores this, but the SDK requires it
  endpoint: process.env.S3_ENDPOINT || 'http://localhost:9000',
  credentials: {
    accessKeyId: process.env.S3_ACCESS_KEY || 'minioadmin',
    secretAccessKey: process.env.S3_SECRET_KEY || 'your-strong-password-here',
  },
  forcePathStyle: true,  // Required for MinIO
});

// Upload a file
async function uploadFile(bucket, key, body) {
  await s3.send(new PutObjectCommand({
    Bucket: bucket,
    Key: key,
    Body: body,
    ContentType: 'image/jpeg',
  }));
}

// Generate a presigned URL (user can upload directly)
async function getUploadUrl(bucket, key) {
  const command = new PutObjectCommand({ Bucket: bucket, Key: key });
  return getSignedUrl(s3, command, { expiresIn: 3600 });
}

// Generate a presigned download URL
async function getDownloadUrl(bucket, key) {
  const command = new GetObjectCommand({ Bucket: bucket, Key: key });
  return getSignedUrl(s3, command, { expiresIn: 3600 });
}
```

The critical setting is `forcePathStyle: true`. Without it, the SDK tries virtual-hosted-style URLs (`bucket.localhost:9000`) which won't work with MinIO's default setup.

### Python (boto3)

```python
import boto3
from botocore.client import Config

s3 = boto3.client(
    's3',
    endpoint_url='http://localhost:9000',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='your-strong-password-here',
    config=Config(signature_version='s3v4'),
    region_name='us-east-1',
)

# Upload
s3.upload_file('local-file.pdf', 'uploads', 'reports/q4.pdf')

# Presigned URL
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'uploads', 'Key': 'reports/q4.pdf'},
    ExpiresIn=3600,
)
```

Same deal. Point the endpoint at MinIO and everything works.

## Presigned URLs: The Right Way to Handle Uploads

Don't pipe user uploads through your app server. It's slow, it eats your RAM, and it's a footgun for large files. Instead:

1. Client asks your API for an upload URL
2. Your API generates a presigned PUT URL from MinIO
3. Client uploads directly to MinIO using that URL
4. Client tells your API "hey, I uploaded `uploads/abc123.jpg`"
5. Your API records the file reference in the database

The file never touches your app server. MinIO handles the upload directly. This is the same pattern AWS recommends for S3, and it works identically with MinIO.

For downloads, same idea. Generate a presigned GET URL and redirect the user. No bandwidth through your app.

## Putting It Behind a Reverse Proxy

For production, you want MinIO behind [nginx](https://nginx.org/) with SSL:

```nginx
# MinIO S3 API
server {
    listen 443 ssl http2;
    server_name s3.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/s3.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/s3.yourdomain.com/privkey.pem;

    # Allow large uploads
    client_max_body_size 100M;

    location / {
        proxy_pass http://127.0.0.1:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# MinIO Console (optional, restrict access)
server {
    listen 443 ssl http2;
    server_name minio-console.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/minio-console.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/minio-console.yourdomain.com/privkey.pem;

    # Restrict to your IP
    allow 203.0.113.0/24;
    deny all;

    location / {
        proxy_pass http://127.0.0.1:9001;
        proxy_set_header Host $host;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Don't forget `client_max_body_size`. The default nginx limit is 1MB, which will silently break file uploads and leave you debugging for an hour.

## Backing Up MinIO to Backblaze B2

Self-hosting your storage is great, but you still need offsite backups. The 3-2-1 rule applies here too. I use `mc mirror` to replicate my MinIO data to Backblaze B2:

```bash
# Set up B2 as an mc alias
mc alias set b2 https://s3.us-west-004.backblazeb2.com YOUR_B2_KEY_ID YOUR_B2_APP_KEY

# Mirror your MinIO bucket to B2
mc mirror myminio/uploads b2/my-minio-backup/uploads --overwrite

# Cron it (daily at 4am)
# 0 4 * * * /usr/local/bin/mc mirror myminio/uploads b2/my-minio-backup/uploads --overwrite >> /var/log/minio-backup.log 2>&1
```

Cost: pennies. A 20GB bucket on B2 costs about $0.12/month for storage.

## Creating Service Accounts

Don't give your app the root credentials. Create a dedicated access key:

```bash
# Via mc
mc admin user add myminio app-uploads-user strong-password-here

# Create a policy (read/write to the uploads bucket only)
cat > /tmp/uploads-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::uploads", "arn:aws:s3:::uploads/*"]
    }
  ]
}
EOF

mc admin policy create myminio uploads-readwrite /tmp/uploads-policy.json
mc admin policy attach myminio uploads-readwrite --user app-uploads-user
```

Your app gets the minimum permissions it needs. If those credentials leak, the blast radius is one bucket, not your entire storage.

## Resource Usage

MinIO single-server is light:

- **RAM:** ~200-400MB depending on load
- **CPU:** Minimal for typical indie workloads
- **Disk:** Whatever your data takes, plus ~1% overhead

It'll run fine alongside your apps on a 2GB VPS. I wouldn't recommend it for boxes with less than 2GB total though -- leave room for your actual applications.

## When to NOT Self-Host Storage

Be honest with yourself. If you're storing:

- **> 500GB of data:** Look at Backblaze B2 or [Cloudflare R2](https://www.cloudflare.com/developer-platform/r2/) directly. Disk is cheap on a VPS but not that cheap.
- **Media that needs a CDN:** MinIO doesn't have edge nodes. Put [Cloudflare](https://www.cloudflare.com/) in front of it, or use R2 which comes with the CDN built in.
- **Compliance-sensitive data:** Healthcare, finance, etc. The managed services have certifications you'd spend years getting.

For everything else -- user uploads under a few hundred gigs, app assets, PDF generation, internal tooling -- MinIO on your VPS is the sweet spot. Free, fast, and you own it.

## Cheat Sheet

| Tool/Technique | What It Does | Link |
|---|---|---|
| **MinIO** | S3-compatible object storage you run yourself | [min.io](https://min.io/) |
| **`mc` (MinIO Client)** | CLI for managing buckets and objects across S3-compatible stores | [min.io/docs/minio/linux/reference/minio-mc.html](https://min.io/docs/minio/linux/reference/minio-mc.html) |
| `mc alias set` | Register an S3-compatible endpoint for CLI use | -- |
| `mc mb myminio/bucket` | Create a new bucket | -- |
| `mc mirror` | Sync data between S3-compatible stores (like rsync for buckets) | -- |
| `mc anonymous set download` | Make a bucket publicly readable | -- |
| `mc admin user add` | Create a service account with limited permissions | -- |
| **`forcePathStyle: true`** | Required AWS SDK setting for MinIO (skip virtual-hosted URLs) | -- |
| **Presigned URLs** | Let clients upload/download directly to MinIO, bypass your app server | -- |
| `client_max_body_size` | Nginx setting to allow large uploads (default 1MB is too small) | -- |
| **Backblaze B2** | Cheap offsite backup target for your MinIO data ($0.006/GB/mo) | [backblaze.com/b2](https://www.backblaze.com/b2/cloud-storage.html) |
| **Cloudflare R2** | S3-compatible with built-in CDN, no egress fees | [developers.cloudflare.com/r2](https://developers.cloudflare.com/r2/) |
| **Least-privilege policies** | Give your app only the bucket permissions it needs | -- |
