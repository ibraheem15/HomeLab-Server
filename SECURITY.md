# Security and Disclosure Policy

This repository is documentation-only. It intentionally excludes live credentials, public IP addresses, Tailscale addresses, device identifiers, personal account names, real domain names, and private configuration exports. Generic role names such as `services-vm` are retained.

## Placeholder convention

Angle-bracketed values such as `<SECRET>`, `<SERVICES_VM_LAN_IP>`, and `<FILESYSTEM_UUID>` are placeholders. Store real values in a password manager or in local files excluded by `.gitignore`; do not replace placeholders in committed documentation.

## Files that must never be committed

- `.env` files and API/tunnel tokens
- iSCSI target exports such as `saveconfig.json`, which can contain CHAP credentials
- `/etc/iscsi` node databases
- Tailscale state or authentication material
- SSH/private keys and certificate private keys
- database dumps, application databases, photo/video libraries, or diagnostic archives
- logs that have not been reviewed for tokens, addresses, domains, serial numbers, and usernames

## Reporting a problem

If a committed value appears to be a real secret or personal infrastructure identifier, use GitHub’s private vulnerability-reporting feature when available. Do not quote the suspected secret in a public issue.

Revoking or rotating a leaked credential is the first response. Removing it from the latest commit does not remove it from Git history, forks, caches, or existing clones.
