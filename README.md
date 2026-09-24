# Capstone: Automated Flask App Deployment on AWS with Terraform

## Overview

This project provisions AWS infrastructure with Terraform and deploys a
Dockerized Flask + PostgreSQL application automatically via `cloud-init`,
with zero manual steps after `terraform apply`. Built as the capstone for
a self-directed DevOps / infrastructure-automation learning plan, tying
together Terraform, Docker, Linux, and Git into one working deployment.

## Architecture

- **VPC** (`10.0.0.0/16`) with a public subnet (`10.0.1.0/24`) in
  `eu-central-1a`
- **Internet Gateway** + public route table for outbound/inbound access
- **Security group** (`capstone-app-sg`):
  - Port `5000` (Flask app) open to `0.0.0.0/0`
  - Port `22` (SSH) restricted to a single trusted IP
- **EC2 instance** (`t3.micro`, Ubuntu 24.04 LTS), bootstrapped via
  `user_data.sh` on first boot
- **Two repositories:**
  - [`capstone-project`](.) — this repo; all Terraform infrastructure code
  - [`docker-lab`](https://github.com/arashkarimi6788/docker-lab) — the
    application itself (Flask + Postgres, via `docker-compose`)

  These are intentionally separate: infrastructure and application code
  change on different cadences and are versioned independently, the way
  they typically are on a real team. The tradeoff is that a deploy
  touching the app requires two separate `git push`/`git pull` steps
  (infra vs. app) — worth knowing if you're new to the repo.

## How it works

1. `terraform apply` provisions the VPC, subnet, internet gateway, route
   table, security group, and EC2 instance.
2. On first boot, `user_data.sh` runs automatically:
   - Waits for network connectivity to be ready (retry loop — see
     *Problems solved* below)
   - Installs Docker and the Docker Compose plugin
   - Clones the `docker-lab` repo into `/opt/docker-lab`
   - Runs `docker compose up -d --build`, starting the Postgres and
     Flask containers
3. The app becomes reachable at `http://<instance-public-ip>:5000`

## Deployment

```bash
cd capstone-project
terraform init      # first time only
terraform apply
```

Terraform outputs the instance's public IP and app URL when done:
app_url = "http://<ip>:5000/items"
instance_public_ip = "<ip>"


## Updating the app

App code lives in the separate `docker-lab` repo, not here:

```bash
git clone git@github.com:arashkarimi6788/docker-lab.git
cd docker-lab
# make your changes to app.py
git add app.py
git commit -m "describe the change"
git push
```

Then pull and rebuild on the running instance:

```bash
ssh -i ~/.ssh/aws-lab-key.pem ubuntu@<instance-ip>
cd /opt/docker-lab
sudo git pull
sudo docker compose up -d --build
```

Confirm the change took effect by checking that `COPY app.py .` in the
build output does *not* say `CACHED` — if it does, the rebuild picked up
no changes, usually meaning the push didn't reach the repo the instance
actually pulls from.

## Teardown

```bash
terraform destroy
```

## Endpoints

- `GET /health` — health check, returns `{"status": "ok"}`
- `GET /items` — returns the list of items (JSON array)

## Problems solved along the way

- **Race condition on boot:** the original `user_data.sh` could attempt
  the Docker install before the network interface was ready, causing
  silent failures on some boots. Fixed by adding a network-readiness
  retry loop before any install steps run.
- **SSH "connection refused" after `terraform apply`:** diagnosed by
  checking `aws ec2 describe-instance-status` (confirmed instance
  healthy) and `aws ec2 get-console-output` (confirmed `cloud-init` was
  still mid-boot) rather than assuming a broken security group.
- **Stale Docker build cache masking a real bug:** a code change wasn't
  showing up after rebuilding the container. Traced through `git status`,
  `git remote -v` on both the local machine and the instance, and found
  the instance's `/opt/docker-lab` was cloned from a *different* GitHub
  repo (`docker-lab`) than the one being edited locally
  (`capstone-project`) — Docker's build cache was correctly reporting no
  changes to a file it had never actually received.

## Tech stack

Terraform · AWS (VPC, EC2, Security Groups) · Docker & Docker Compose ·
Flask · PostgreSQL · `cloud-init` / `user_data`
