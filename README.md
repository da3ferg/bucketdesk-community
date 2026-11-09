# BucketDesk Community Edition

A file workspace over the Amazon S3 buckets you already own, running on your own machine. Browse,
search and preview your S3 files in the browser, and give a teammate read access without handing
out AWS logins. Your files stay in your bucket.

```bash
git clone https://github.com/da3ferg/bucketdesk-community.git
cd bucketdesk-community
docker compose up -d
```

Open **http://localhost:8080** and create your account. That is the whole install.

---

## Quick start

**You need:** Docker with Docker Compose (Docker Desktop on macOS or Windows, Docker Engine on Linux),
about 2 GB of free memory, and an Intel/AMD or ARM processor.

1. **Start it.** In this folder, run:

    ```bash
    docker compose up -d
    ```

    The first start downloads the images and prepares the database. It is ready in about a minute,
    when this shows `healthy` next to `web`:

    ```bash
    docker compose ps
    ```

2. **Create your account.** Open http://localhost:8080 and sign up. The first account owns the
   workspace.

3. **Confirm your email.** BucketDesk ships with a built-in inbox. Open http://localhost:8025, open
   the "Confirm your email" message and click the link.

4. **Connect your bucket.** Follow [Connect your bucket](#connect-your-bucket) below. It takes about
   five minutes in the AWS console.

Every setting has a working default. The encryption key and database are created on first start and
kept in Docker volumes, so your workspace survives restarts and upgrades.

## What is included

| | Community Edition |
| --- | --- |
| Buckets | 1 |
| People | 1 owner and 1 viewer |
| Browse folders in list or grid view | Yes |
| Search file names | Yes |
| Preview images, PDFs, text and code | Yes |
| File details and S3 metadata | Yes |
| BucketDesk tools for S3 | Yes |
| Price | Free |

Uploads, sharing links, office document previews, video streaming, document chat, archive tools and
larger teams are part of the paid plans. Paste a licence under **Settings, Licence** to turn them on;
see [Upgrade](#upgrade).

## Connect your bucket

BucketDesk reaches your bucket through AWS IAM roles in your own account. You create them once, with
two CloudFormation stacks, and BucketDesk opens your files through short-lived role sessions.

**1. Give this install an AWS identity (once).**

1. Sign in to the AWS console for the account that owns your bucket, and open
   [CloudFormation, Create stack](https://console.aws.amazon.com/cloudformation/home#/stacks/create).
2. Choose **Upload a template file**, pick [`aws-access.yaml`](aws-access.yaml) from this folder, name
   the stack `bucketdesk-access` and create it. Tick the box that acknowledges IAM resources.
3. When the stack shows `CREATE_COMPLETE`, open its **Outputs** tab.
4. Copy the `CredentialsFileCommand` value and run it in this folder. It needs the
   [AWS CLI](https://aws.amazon.com/cli/) signed in to the same account, and writes `aws/credentials`.
5. Create a file named `.env` in this folder with the `PrincipalArn` value:

    ```dotenv
    AWS_CUSTOMER_ACCESS_PRINCIPAL_ARN=arn:aws:iam::123456789012:role/BucketDeskSelfHosted-us-east-1
    AWS_DEFAULT_REGION=us-east-1
    ```

6. Apply it:

    ```bash
    docker compose up -d
    ```

The key in `aws/credentials` can do one thing: assume the `BucketDeskSelfHosted` role. That role can
only assume the per-bucket roles you approve in the next step.

**2. Connect the bucket (in BucketDesk).**

In BucketDesk, open **Connections, Connect your first bucket**, enter the bucket name and your
12-digit account ID, and choose **Continue to AWS to review**. A CloudFormation page opens in your own
console with a read-only role for that bucket. Create it, return to BucketDesk, and your files appear.

## Settings

Put any of these in a `.env` file next to `compose.yaml`, then run `docker compose up -d`.
[`.env.example`](.env.example) lists them all.

| Setting | Default | What it does |
| --- | --- | --- |
| `BUCKETDESK_PORT` | `8080` | Port the app listens on |
| `APP_URL` | `http://localhost:8080` | Address people open; used in email links |
| `MAILPIT_PORT` | `8025` | Port of the built-in inbox |
| `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_SCHEME`, `MAIL_FROM_ADDRESS` | built-in inbox | Your own mail server, so invitations reach real inboxes |
| `AWS_CUSTOMER_ACCESS_PRINCIPAL_ARN` | empty | `PrincipalArn` from the `bucketdesk-access` stack |
| `AWS_DEFAULT_REGION` | `us-east-1` | Region of that stack |
| `DB_PASSWORD` | `bucketdesk` | Database password; set it before the first start |
| `SELF_HOSTED_LICENCE` | empty | Licence key for paid features |

### Run it for your team

To let a colleague open it from another computer, give the install an address on your network and
HTTPS:

1. Put a reverse proxy with a certificate in front of port 8080 (Caddy, nginx, Traefik or a cloud
   load balancer).
2. In `.env`, set `APP_URL=https://bucketdesk.yourcompany.internal` and `SESSION_SECURE_COOKIE=true`.
3. Set the `MAIL_*` settings to your mail server, so the invitation reaches your colleague.
4. Run `docker compose up -d`. The built-in inbox stays reachable only from the machine running BucketDesk.

## Everyday commands

```bash
docker compose ps          # status
docker compose logs -f web # app logs
docker compose stop        # stop, keeping your data
docker compose up -d       # start again
```

## Upgrade

**New release.** Pull this repository and restart. Your data and key stay in their volumes.

```bash
git pull
docker compose pull
docker compose up -d
```

**Paid features.** Create a free 30-day trial or buy a licence from your account at
[app.bucketdesk.com](https://app.bucketdesk.com/settings/self-hosted) (Settings, Self-hosted), then
paste it in your install under **Settings, Licence**. The licence is checked on your machine.

## Back up

Everything BucketDesk keeps lives in the `postgres-data` and `secrets` volumes; your files stay in S3.

```bash
docker compose exec postgres pg_dump -U bucketdesk bucketdesk > bucketdesk-backup.sql
```

Keep `aws/credentials` and your `.env` with the backup.

## Remove

```bash
docker compose down -v
```

This deletes the workspace database and key. Your S3 files are untouched. To remove AWS access too,
delete the `bucketdesk-access` stack and the per-bucket stacks in CloudFormation.

## Troubleshooting

**Port 8080 is already in use.** Set another port in `.env`, for example `BUCKETDESK_PORT=8090`, and
run `docker compose up -d`. Email links follow the new port.

**`web` is not healthy.** Read `docker compose logs web`. The first start waits for the database and
creates its tables, which can take a minute on a slow disk.

**No confirmation email.** Open the built-in inbox at http://localhost:8025. If you set your own
`MAIL_*` settings, check `docker compose logs worker`.

**The bucket connection fails.** Check that `.env` has the `PrincipalArn` from the
`bucketdesk-access` stack, that `aws/credentials` exists, and that you ran `docker compose up -d`
after adding both.

## About

BucketDesk is a workspace for the files your company keeps in Amazon S3. The hosted version and paid
plans are at [bucketdesk.com](https://bucketdesk.com). Questions and bug reports:
[open an issue](https://github.com/da3ferg/bucketdesk-community/issues).

The files in this repository are released under the [MIT licence](LICENSE). The BucketDesk
application image they run is free to use as Community Edition under the
[BucketDesk terms](https://bucketdesk.com/terms).
