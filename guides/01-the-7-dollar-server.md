---
title: "The $7/month server: set up a VPS for self-hosting in 30 minutes"
description: "A safe, boring, repeatable setup for one small Linux server that can run every app on this site."
last_checked: 2026-10-30
tested_on: "Ubuntu 24.04 LTS (SSH hardening and packages checked in an Ubuntu 24.04 container)"
---

> **Disclosure:** this guide has no affiliate links today. If we add any later (for example to server providers), they'll be marked here, and we recommend the same options whether or not they pay us.

Most small-business software is a monthly bill for something a $5 to $10 server can run. Before you replace any of it, you need one server set up properly. This guide does that once, so every other guide on the site can start with "on your server, run…".

By the end you will have:

- a Linux server you log into with a key, not a password
- a firewall that only lets in web traffic and SSH
- security updates that install themselves
- Docker, which every app guide on this site uses

Time: about 30 minutes. Cost: $5 to $10 a month.

## 1. Pick a server

For running 3 to 6 small business apps (scheduling, analytics, a password manager, uptime monitoring), you want:

| Spec | Minimum | Comfortable |
| --- | --- | --- |
| CPU | 1 vCPU | 2 vCPU |
| Memory | 2 GB | 4 GB |
| Disk | 40 GB SSD | 80 GB SSD |
| OS | Ubuntu 24.04 LTS | Ubuntu 24.04 LTS |

Most providers sell this for $5 to $12 a month. Good options include Hetzner (cheapest in Europe), DigitalOcean and Vultr (simple dashboards, many regions) and Hostinger (cheap long-term plans). Pick the region closest to your customers. A full comparison of these providers is coming soon.

When the provider asks for an SSH key, give it one (next step). Choose **Ubuntu 24.04 LTS**.

## 2. Make an SSH key (on your own computer)

An SSH key replaces your password with a file only your computer has. On macOS, Linux or Windows 10+ (PowerShell), run:

```bash
ssh-keygen -t ed25519 -C "you@yourbusiness.com"
```

Press Enter to accept the default location and set a passphrase. Then print the public half:

```bash
cat ~/.ssh/id_ed25519.pub
```

Paste that one line into your provider's "SSH keys" box when creating the server. Never share the file without `.pub`.

## 3. Log in and create your own user

Your provider shows the server's IP address. Log in as root once:

```bash
ssh root@YOUR_SERVER_IP
```

Create a normal user (we'll call it `deploy`), give it admin rights, and copy your key to it:

```bash
adduser deploy
usermod -aG sudo deploy
rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy
```

Open a **second** terminal and check that the new user works before you go further:

```bash
ssh deploy@YOUR_SERVER_IP
sudo whoami   # should print: root
```

## 4. Turn off password and root logins

Still as `deploy`, write a small settings file:

```bash
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf > /dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
EOF
sudo sshd -t && sudo systemctl restart ssh
```

`sshd -t` checks the file first, so a typo can't lock you out. On Ubuntu 24.04 the service is called `ssh`, not `sshd`.

Keep your current session open and test a fresh login from another terminal. If it works, you're done with SSH.

## 5. Firewall: allow only SSH and web traffic

```bash
sudo apt update
sudo apt install -y ufw
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

Allow `OpenSSH` **before** `enable`, or you'll cut off your own connection.

One catch worth knowing: ports Docker publishes can bypass ufw. That's why every compose file on this site binds app ports to `127.0.0.1` and lets one web server (Caddy) face the internet on 80 and 443.

## 6. Automatic security updates

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades   # choose "Yes"
```

This installs Ubuntu's security fixes every day. It won't do big version upgrades, so nothing changes under you unexpectedly.

## 7. Install Docker

Docker runs each app in its own box, so apps can't break each other and you can remove one cleanly. Use Docker's official install script:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker deploy
```

Log out and back in, then check:

```bash
docker run --rm hello-world
docker compose version
```

Docker's own docs [describe this script and the manual alternative](https://docs.docker.com/engine/install/ubuntu/). Read the script first if you prefer (`less get-docker.sh`).

## 8. Point a domain at the server

In your domain's DNS settings, add an **A record** for each app you'll host, pointing at your server's IP, for example:

| Name | Type | Value |
| --- | --- | --- |
| status | A | YOUR_SERVER_IP |
| cal | A | YOUR_SERVER_IP |

Caddy, used in every app guide, gets free HTTPS certificates for these names automatically.

## What's next

Your server is ready. Good first apps:

- [Uptime Kuma](./02-uptime-kuma.md): free monitoring that tells you when your website goes down
- [Cal.com](./03-calcom.md): replace Calendly
- [Vaultwarden](./05-vaultwarden.md): a team password manager
- [One Caddy for all your apps](./07-caddy.md): run several apps on one server

Before you put anything important on it, set up [nightly backups](./08-backups.md). A server without backups is a gamble.

---

*Last checked October 30, 2026 on Ubuntu 24.04 LTS. Found a step that no longer works? [Open an issue](https://github.com/whatnowhater-oss/own-your-stack/issues) and we'll fix it.*
