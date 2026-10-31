---
title: "The $7/month server: set up a VPS for self-hosting in 30 minutes"
description: "A safe, boring, repeatable setup for one small Linux server that can run every app on this site."
last_checked: 2026-10-31
tested_on: "Ubuntu 24.04 LTS on a real Vultr server (1 vCPU, 1 GB), steps 2 to 7 run end to end from a Windows PC, Oct 31 2026"
---

> **Disclosure:** this guide has one affiliate link: the Vultr link below. If you sign up through it, Vultr may pay us a referral fee at no cost to you. We recommend the same providers either way, and the others are listed with plain links.

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

Most providers sell this for $5 to $12 a month. Good options include Hetzner (cheapest in Europe), DigitalOcean and <a href="https://www.vultr.com/?ref=9932432" rel="sponsored nofollow">Vultr</a> (affiliate link; simple dashboards, many regions) and Hostinger (cheap long-term plans). Pick the region closest to your customers. For a first server, any of them is fine: pick on price and region, since all four offer Ubuntu 24.04 servers with the specs above. See [current prices compared](./09-which-vps.md).

When the provider asks for an SSH key, give it one (next step). Choose **Ubuntu 24.04 LTS**.

Check the order form before you pay. When we created a server at Vultr in October 2026, the form's defaults included choices you probably don't want:

- **Plan type:** pick **Shared CPU**. Dedicated CPU costs several times more.
- **Operating system:** the form picked a newer, non-LTS Ubuntu. Pick **24.04 LTS** explicitly, because LTS versions get 5 years of security updates.
- **Auto backups:** these were switched on, at extra cost each month. You can leave them on as a second layer, but [your own backups](./08-backups.md) are what this site relies on.
- **SSH key:** the form had no way to add a new key. Add the key under **Account → SSH Keys** first, then open the order form, because reloading the form loses what you picked.

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

**On Windows:** if PowerShell says `ssh` is not recognized, the OpenSSH client isn't installed. Add it under **Settings → System → Optional features → OpenSSH Client**, or use **Git Bash** if Git is installed. On our test PC, `ssh-keygen` worked in PowerShell but `ssh` didn't, so check both.

## 3. Log in and create your own user

Your provider shows the server's IP address. Log in as root once:

```bash
ssh root@YOUR_SERVER_IP
```

The first time you connect, SSH asks *"Are you sure you want to continue connecting (yes/no)?"*. Type `yes`. It's just confirming that this is a server you haven't connected to before.

Create a normal user (we'll call it `deploy`), give it admin rights, and copy your key to it:

```bash
adduser deploy
usermod -aG sudo deploy
rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy
```

`adduser` asks you to set a password for `deploy`. **Pick a strong one and save it in your password manager.** You won't use it to log in, but `sudo` asks for it whenever you run an admin command. Press Enter to skip the name and phone questions.

Open a **second** terminal and check that the new user works before you go further:

```bash
ssh deploy@YOUR_SERVER_IP
sudo whoami   # should print: root
```

## 4. Turn off password and root logins

Still as `deploy`, write a small settings file:

```bash
sudo tee /etc/ssh/sshd_config.d/00-hardening.conf > /dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
EOF
sudo sshd -t && sudo systemctl restart ssh
```

`sshd -t` checks the file first, so a typo can't lock you out. On Ubuntu 24.04 the service is called `ssh`, not `sshd`.

**The file name matters.** Many providers' Ubuntu images put their own file in the same folder, such as `50-cloud-init.conf`, and it turns password logins back **on**. SSH reads the files in name order and keeps the first value it finds, so your file's name must start with `00` to take priority. We found this on a real Vultr server: with the file named `99-hardening.conf`, password logins still worked. Check that the setting took effect:

```bash
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin'
```

Both lines must end in `no`. If either says `yes`, check the file name and look for other files in `/etc/ssh/sshd_config.d/`.

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

Allow `OpenSSH` **before** `enable`, or you'll cut off your own connection. `ufw enable` warns that it *"may disrupt existing ssh connections"* and asks `(y|n)`. Type `y`. Your session stays connected because you allowed SSH first.

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

*Tested October 31, 2026 on a real Vultr server: 1 vCPU and 1 GB of memory, running Ubuntu 24.04 LTS, set up from a Windows PC. Steps 2 to 7 worked as written once the fixes above were in. After step 4, root and password logins were both refused, ufw allowed only 22, 80 and 443, unattended-upgrades was active, and Docker ran without sudo. Step 8 wasn't tested with a real domain, but HTTPS through Caddy worked on a test hostname. Found a step that no longer works? [Open an issue](https://github.com/whatnowhater-oss/own-your-stack/issues) and we'll fix it.*
