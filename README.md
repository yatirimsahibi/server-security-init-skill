# Server Security Init Skill

A Codex skill for safely initializing and hardening fresh Ubuntu/Debian SSH servers.

The skill guides an agent through a staged server bootstrap flow: selecting or generating a local SSH key, requiring public-key login before remote changes, creating a non-root sudo user, changing the SSH port, disabling root/password login, configuring UFW, configuring fail2ban, updating local SSH config, and verifying the operator is not locked out.

## What It Covers

- Public-key bootstrap without putting root passwords into commands, logs, or config files.
- Staged SSH hardening with a verified non-root sudo login before disabling old access paths.
- UFW default-deny inbound policy with only explicitly requested ports allowed.
- Fail2ban `sshd` jail setup with management IPs in `ignoreip`.
- Debian 12/systemd `ssh.socket` checks so socket activation does not silently keep SSH on the old port.
- Verification commands that prefer server-side truth such as `ss -ltnp`, `sshd -T`, `systemctl`, and `ufw status`.
- Recovery checks for fail2ban bans, firewall mistakes, and SSH listener mismatches.

## Repository Layout

```text
server-security-init-skill/
  server-security-init/
    SKILL.md
    agents/openai.yaml
    references/
      ubuntu-openssh-ufw-fail2ban.md
      verification-checklist.md
  scripts/
    validate_skill.py
  .github/workflows/validate.yml
  README.md
  LICENSE
  SECURITY.md
```

The installable skill is the `server-security-init/` directory. The repository-level files are for humans, validation, and open-source maintenance.

## Installation

Clone this repository, then copy the skill directory into your Codex skills directory.

PowerShell:

```powershell
Copy-Item -Recurse .\server-security-init "$env:USERPROFILE\.codex\skills\server-security-init"
```

Bash:

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
cp -R ./server-security-init "${CODEX_HOME:-$HOME/.codex}/skills/server-security-init"
```

Restart Codex after installing or updating the skill.

## Usage

Ask Codex to use the skill when initializing a fresh server, for example:

```text
Use $server-security-init to initialize my new Ubuntu server.
```

The skill is intentionally conservative. It should ask for bootstrap facts before touching the server and should stop if public-key SSH login cannot be verified.

## Safety Notes

- Review the generated commands before running them on production systems.
- Keep an existing working SSH session open while changing SSH or firewall settings.
- Do not paste private keys or root passwords into prompts, scripts, command lines, or repository files.
- Provider consoles, rescue panels, and VNC access are still required recovery paths for mistakes outside SSH reachability.
- This project is guidance for AI-assisted administration, not a substitute for understanding the commands being run.

## Validation

Run the repository validator before publishing changes:

```bash
python scripts/validate_skill.py
```

The validator checks required files, frontmatter, and common secret/personal-data patterns.

## Supported Targets

This skill is written for Ubuntu/Debian systems using OpenSSH, systemd, UFW, and fail2ban. Other distributions, firewall systems, init systems, and SSH implementations need careful adaptation.

## License

MIT License. See [LICENSE](LICENSE).
