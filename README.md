# 𝐓𝐢𝐭𝐚𝐧𝐂𝐥𝐨𝐮𝐝™ | VPS Bot

A Discord-managed Docker VPS service with an environment-configurable plan:

- 8 GB RAM, 4 CPU cores, and 60 GB displayed storage by default
- one container per eligible Discord user; owners bypass the per-user quota
- owner-only `/create_vps` provisioning with a selected member and larger RAM, disk, and CPU values
- owner access through the configured owner role or owner IDs
- self-service deployment restricted to the configured user role
- premium DM embeds and custom Discord controls

## Ubuntu setup

Docker is required in addition to Git and Python. Run the bot as a user that can access the Docker daemon.

```bash
sudo apt update
sudo apt install -y git python3-venv docker.io
sudo systemctl enable --now docker

git clone <your-modified-repository-url> /root/Private1
cd /root/Private1

python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
nano .env
python3 titancloud_bot.py
```

The requirements include Discord's optional voice encryption helper so startup does not emit a voice-support warning, although TitanCloud itself does not use voice features.

Add your Discord bot token to `.env`. The supplied IDs are already the defaults, but keeping them in `.env` makes the deployment explicit. Never commit `.env` or share the token.

Important: use the correct filename. Run `cp .env.example .env`, then edit `nano .env`. A command such as `nano .env.examplec` contains a typo, creates the wrong file, and will not configure the bot.

To obtain a valid token, open the Discord Developer Portal, select the application, open **Bot**, choose **Reset Token**, and copy the new token immediately. Set it in `.env` without a `Bot ` prefix:

```env
DISCORD_TOKEN=paste_the_new_token_here
```

A `401 Unauthorized` or `Improper token has been passed` error means Discord rejected that token. Reset it again, update `.env`, and restart the process.

In the Discord Developer Portal, enable the **Server Members Intent** and **Message Content Intent** for the bot. Invite it with the `bot` and `applications.commands` scopes.

## Access model

`/deploy` creates an Ubuntu 22.04 container for the invoking member using the normal plan. Regular members must have role `1536277694927474720` and may own one VPS.

Owners can create a larger plan for a selected member with `/create_vps target:@member ram:16 disk:120 core:8`. The target, RAM, disk, and core options are required. Custom plans cannot be smaller than the normal 8 GB RAM, 4-core, 60 GB plan. Their default upper limits are 256 GB RAM, 64 cores, and 2000 GB displayed disk, controlled by `MAX_CUSTOM_MEMORY_GB`, `MAX_CUSTOM_CPU_CORES`, and `MAX_CUSTOM_DISK_GB` in `.env`. Owner deployments bypass the per-user quota but always respect the global container capacity. Deployments intentionally do not run `docker build`, which keeps them compatible with nested VPS hosts that reject overlay mounts.

`/manage` automatically opens the invoking member's only VPS. When the member owns multiple VPS instances, use `/manage vps_id:<id>`. The control panel is also delivered by DM and includes Start, Stop, Restart, current-OS Reinstall, Fresh SSH, and Transfer buttons.

Every Fresh SSH click is serialized per VPS: the bot closes all previous tmate processes and sockets before creating one replacement, stores the new session, and sends it directly to the clicking user's DMs. Rapid repeated clicks cannot race with each other; each later click intentionally revokes the session created by the earlier click. The DM contains only the private SSH command—no username, password, root password, or access token. The user must allow DMs from server members for delivery.

To log in, run `/manage`, click **Fresh SSH**, open the bot's DM, copy the complete `ssh ...` command, paste it into a terminal, and press Enter. No separate login name or password is used.

Control-panel buttons are persistent: they do not expire after five minutes, and the bot restores saved VPS controls after a restart. The backup command writes non-executable `titancloud_backup.json` data instead of an unsafe pickle file.

Owners are recognized by role `1426555185454649447` or either configured owner ID.

RAM, CPU, displayed disk allocation, custom-plan ceilings, per-user quota, image, Docker network, and global capacity can be configured through `.env`. The normal-plan defaults are `DEFAULT_MEMORY_GB=8`, `DEFAULT_CPU_CORES=4`, and `DEFAULT_DISK_GB=60`. Restart the bot after changing `.env`.

The disk value is allocation metadata shown by the bot; Docker named volumes do not enforce a portable size quota. Enforce physical storage quotas on the Docker host when required.

## Network and firewall policy

TitanCloud does not install, initialize, or run UFW, iptables, or ip6tables inside managed containers. Containers share the host kernel, so their reachable services must be controlled with Docker port publishing and the Docker host's firewall.

The managed image also does not install a nested Docker daemon. This prevents container-local Docker startup from trying to create firewall chains; Docker lifecycle and network policy remain on the TitanCloud host.

Do not add `ufw enable`, `iptables`, or writes to `/etc/ufw` to container setup commands. TitanCloud access uses the rotating private tmate SSH command and does not require publishing container port 22. If another service needs a host port, publish only that port through Docker and apply public access restrictions on the Docker host or upstream provider firewall.

## Keep the bot running

Running the bot directly ties it to your SSH terminal. Install the included systemd service so it survives logout and automatically restarts after failures:

```bash
cd /root/Private1
sudo cp titancloud-bot.service /etc/systemd/system/titancloud-bot.service
sudo systemctl daemon-reload
sudo systemctl enable --now titancloud-bot
sudo systemctl status titancloud-bot
```

Follow live logs with:

```bash
sudo journalctl -u titancloud-bot -f
```

## Push to GitHub

Create an empty repository in your GitHub account, then run these commands from the `VPS_bot` project directory. Replace the repository URL with your own.

```bash
# The upstream repository tracked .env. Remove only its Git index entry;
# the local deployment file remains on disk and is protected by .gitignore.
git rm --cached .env

git add titancloud_bot.py titancloud-bot.service requirements.txt README.md .gitignore .env.example
git commit -m "Add premium role-based VPS deployment"

git branch -M main
git remote set-url origin https://github.com/YOUR_USERNAME/YOUR_REPOSITORY.git
git push -u origin main
```

If Git asks you to authenticate over HTTPS, use a GitHub personal access token or sign in with GitHub CLI. Do not force-add `.env`. If the token-shaped value in `.env.example` is ever replaced with a real token, remove it from that file before committing and rotate the exposed token immediately.
