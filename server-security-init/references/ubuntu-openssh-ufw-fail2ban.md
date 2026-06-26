# Ubuntu/Debian OpenSSH + UFW + Fail2ban Flow

Use these command patterns after collecting the parameters in `SKILL.md`. Adjust variable values explicitly; do not silently use placeholder values.

## Optional Local Keypair

Use this when the user has no existing SSH key for the server or wants a dedicated per-server key. Recommended naming is the server alias, for example:

```text
~/.ssh/ALIAS
C:\Users\USER\.ssh\ALIAS
```

Do not overwrite existing key files. Check first:

```powershell
Test-Path "IDENTITY_FILE"
Test-Path "IDENTITY_FILE.pub"
```

or on POSIX shells:

```bash
test -e "$IDENTITY_FILE" || test -e "$IDENTITY_FILE.pub"
```

If either file exists, ask the user whether to reuse that key or choose another `local_key_name`.

Generate an ed25519 keypair after the path and passphrase policy are clear:

```powershell
ssh-keygen -t ed25519 -f "IDENTITY_FILE" -C "LOCAL_KEY_COMMENT"
```

For an intentionally empty passphrase, use:

```powershell
ssh-keygen -t ed25519 -f "IDENTITY_FILE" -C "LOCAL_KEY_COMMENT" -N ""
```

Then verify and use the generated public key:

```powershell
ssh-keygen -lf "IDENTITY_FILE.pub"
Get-Content "IDENTITY_FILE.pub"
```

Set:

```text
identity_file = IDENTITY_FILE
public_key_source = IDENTITY_FILE.pub
```

Never print or transmit the private key. Only upload the `.pub` content to the server.

## Root Public-Key Bootstrap

Use this as the default path when the user only has an IP and root password. Do not automate the password login by default. Instead, provide the public key and ask the user to install it once using the provider console, Tabby, VNC, or manual SSH password login.

Show the user this remote command with the actual public key substituted:

```bash
mkdir -p /root/.ssh
chmod 700 /root/.ssh
grep -qxF 'PUBLIC_KEY_HERE' /root/.ssh/authorized_keys 2>/dev/null || echo 'PUBLIC_KEY_HERE' >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

Then verify root public-key login before making any remote changes:

```powershell
ssh -o BatchMode=yes -o PreferredAuthentications=publickey -i "IDENTITY_FILE" -p INITIAL_PORT INITIAL_USER@TARGET_HOST "hostname; id"
```

Expected: the command succeeds and `id` shows the initial user, usually `uid=0(root)`.

If this verification fails, stop. Do not install packages, change SSH configuration, or modify firewall rules.

Password-based bootstrap is an advanced exception. Use it only if the user explicitly asks for it and the available method does not expose the password in command-line arguments, logs, or saved config.

## Preflight

Run read-only checks first:

```bash
hostname
date
id
uname -a
command -v sudo || true
command -v ufw || true
command -v fail2ban-client || true
sshd -T | egrep '^(port|permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication) '
ss -ltnp | egrep 'ssh|sshd|systemd|:22\b' || true
systemctl is-enabled ssh.socket ssh.service 2>/dev/null || true
systemctl is-active ssh.socket ssh.service 2>/dev/null || true
systemctl cat ssh.socket 2>/dev/null || true
ufw status || true
```

Check current keys:

```bash
ls -ld /root/.ssh /home/*/.ssh 2>/dev/null || true
ls -l /root/.ssh/authorized_keys /home/*/.ssh/authorized_keys 2>/dev/null || true
```

## Stage 1: Create Verified Admin Path

Install basics:

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y sudo ufw
```

Create the admin user and install a public key:

```bash
id "$ADMIN_USER" >/dev/null 2>&1 || useradd -m -s /bin/bash "$ADMIN_USER"
mkdir -p "/home/$ADMIN_USER/.ssh"
chmod 700 "/home/$ADMIN_USER/.ssh"
printf '%s\n' "$PUBLIC_KEY" > "/home/$ADMIN_USER/.ssh/authorized_keys"
chmod 600 "/home/$ADMIN_USER/.ssh/authorized_keys"
chown -R "$ADMIN_USER:$ADMIN_USER" "/home/$ADMIN_USER/.ssh"
```

Configure passwordless sudo:

```bash
printf '%s ALL=(ALL) NOPASSWD: ALL\n' "$ADMIN_USER" > "/etc/sudoers.d/$ADMIN_USER-nopasswd"
chmod 0440 "/etc/sudoers.d/$ADMIN_USER-nopasswd"
visudo -c -f "/etc/sudoers.d/$ADMIN_USER-nopasswd"
```

Back up and add a temporary SSH drop-in that keeps both old and new ports:

```bash
stamp=$(date +%Y%m%d-%H%M%S)
cp -a /etc/ssh/sshd_config "/etc/ssh/sshd_config.bak.$stamp"
mkdir -p /etc/ssh/sshd_config.d
cat > /etc/ssh/sshd_config.d/99-bootstrap-ports.conf <<EOF
Port $INITIAL_PORT
Port $NEW_SSH_PORT
PubkeyAuthentication yes
EOF
sshd -t
```

Temporarily allow both ports:

```bash
ufw --force reset
ufw default deny incoming
ufw default allow outgoing
ufw allow "$INITIAL_PORT/tcp"
ufw allow "$NEW_SSH_PORT/tcp"
ufw --force enable
if systemctl list-unit-files --type=socket 2>/dev/null | awk '{print $1}' | grep -qx 'ssh.socket'; then
  systemctl disable --now ssh.socket || true
fi
systemctl enable --now ssh.service 2>/dev/null || systemctl enable --now ssh 2>/dev/null || true
systemctl restart ssh.service 2>/dev/null || systemctl restart ssh 2>/dev/null || systemctl restart sshd
ss -ltnp | grep sshd
```

Before continuing, verify from a separate command:

```bash
ssh -p "$NEW_SSH_PORT" "$ADMIN_USER@$TARGET_HOST" "id; sudo -n whoami"
```

Expected: user is `ADMIN_USER`, sudo output is `root`.

## Stage 2: Final SSH and UFW State

Do not run this stage until the new admin path is verified.

Disable earlier active directives in main and drop-in configs before writing the final policy:

```bash
stamp=$(date +%Y%m%d-%H%M%S)
cp -a /etc/ssh/sshd_config "/etc/ssh/sshd_config.bak.final.$stamp"
if ls /etc/ssh/sshd_config.d/*.conf >/dev/null 2>&1; then
  mkdir -p "/root/ssh-configd-backup-final-$stamp"
  cp -a /etc/ssh/sshd_config.d/*.conf "/root/ssh-configd-backup-final-$stamp/"
fi

sed -i -E 's/^([[:space:]]*)(Port|PermitRootLogin|PasswordAuthentication|PubkeyAuthentication|KbdInteractiveAuthentication|ChallengeResponseAuthentication)([[:space:]].*)/# disabled by server-security-init: \1\2\3/' /etc/ssh/sshd_config
if ls /etc/ssh/sshd_config.d/*.conf >/dev/null 2>&1; then
  sed -i -E 's/^([[:space:]]*)(Port|PermitRootLogin|PasswordAuthentication|PubkeyAuthentication|KbdInteractiveAuthentication|ChallengeResponseAuthentication)([[:space:]].*)/# disabled by server-security-init: \1\2\3/' /etc/ssh/sshd_config.d/*.conf
fi

cat > /etc/ssh/sshd_config.d/99-security-init.conf <<EOF
Port $NEW_SSH_PORT
PubkeyAuthentication yes
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
EOF

sshd -t
if systemctl list-unit-files --type=socket 2>/dev/null | awk '{print $1}' | grep -qx 'ssh.socket'; then
  systemctl disable --now ssh.socket || true
fi
systemctl enable --now ssh.service 2>/dev/null || systemctl enable --now ssh 2>/dev/null || true
systemctl restart ssh.service 2>/dev/null || systemctl restart ssh 2>/dev/null || systemctl restart sshd
ss -ltnp | grep sshd
```

On Debian 12 and similar systemd hosts, `ssh.socket` can keep listening on port 22 even when `sshd_config` says a different `Port`. Treat `ss -ltnp` as the truth. If socket activation is intentionally used instead of `ssh.service`, update the socket unit `ListenStream` values explicitly and verify them with `systemctl cat ssh.socket`.

Set final firewall rules:

```bash
ufw --force reset
ufw default deny incoming
ufw default allow outgoing
ufw allow "$NEW_SSH_PORT/tcp"
# Add any explicitly requested service ports here, one per rule.
ufw --force enable
ufw status
```

## Stage 3: Fail2ban

Before enabling fail2ban, build `IGNORE_IPS` from:

- loopback: `127.0.0.1/8 ::1`
- current admin egress IP
- usual jump hosts or proxy/VPS exits used for management

Install and configure:

```bash
apt-get install -y fail2ban
mkdir -p /etc/fail2ban/jail.d
if [ -f /etc/fail2ban/jail.d/sshd.local ]; then
  cp -a /etc/fail2ban/jail.d/sshd.local "/etc/fail2ban/jail.d/sshd.local.bak.$(date +%Y%m%d-%H%M%S)"
fi

cat > /etc/fail2ban/jail.d/sshd.local <<EOF
[DEFAULT]
ignoreip = $IGNORE_IPS
banaction = ufw

[sshd]
enabled = true
backend = systemd
port = $NEW_SSH_PORT
maxretry = $FAIL2BAN_MAXRETRY
findtime = $FAIL2BAN_FINDTIME
bantime = $FAIL2BAN_BANTIME
EOF

systemctl enable --now fail2ban
systemctl restart fail2ban
fail2ban-client status sshd
```

If a management IP is banned:

```bash
fail2ban-client set sshd unbanip "$MANAGEMENT_IP"
```

Then fix `ignoreip` and restart fail2ban.

## Local SSH Config Update

Before editing `~/.ssh/config`, copy it to a timestamped backup. Replace only the matching host block or append a new block if none exists:

```sshconfig
Host ALIAS
  HostName TARGET_HOST
  User ADMIN_USER
  Port NEW_SSH_PORT
  IdentityFile IDENTITY_FILE
```

Verify:

```powershell
ssh -G ALIAS | Select-String -Pattern '^(hostname|user|port|identityfile) '
```

Avoid broad regular expressions that can consume following `Host` blocks.

## TUN/Fake-IP Note

When Clash/Mihomo TUN or fake-IP DNS is enabled, local `ping` and `Test-NetConnection` can report misleading success through the virtual interface. Prefer server-side checks:

```bash
ss -ltnp | grep sshd
sshd -T | egrep '^(port|permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication) '
ufw status
```
