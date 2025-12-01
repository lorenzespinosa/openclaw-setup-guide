# Security Hardening Checklist

> Complete checklist for securing an OpenClaw deployment in a legal environment.
> Review this checklist before going live and revisit monthly.

---

## Gateway Security

- [ ] Gateway binds to `loopback` (127.0.0.1) only — never `0.0.0.0`
- [ ] Gateway auth mode set to `token`
- [ ] Gateway token is randomly generated (min 32 characters)
- [ ] Gateway port (3578) is NOT open in firewall
- [ ] No external access to gateway without nginx reverse proxy + SSL

## Agent Tool Control

- [ ] Default agent tools are `allow: ["read"]` only
- [ ] Default agent tools deny `write`, `exec`, `shell`, `system.run`, `browser`
- [ ] Orchestrator cannot exec or shell
- [ ] ops-agent exec is set to `allowlist` mode
- [ ] ops-agent `safeBins` contains only required binaries (e.g., `curl`)
- [ ] ops-agent exec `ask` is set to `always` (interactive) or has audit logging
- [ ] ops-agent `env.inherit` is `false` — no environment leakage
- [ ] research-agent is strictly read-only
- [ ] drafting-agent has no exec or shell access
- [ ] `tools.elevated.enabled` is `false`

## Filesystem Security

- [ ] `tools.fs.workspaceOnly` is `true`
- [ ] `allowedPaths` explicitly set to workspace directory only
- [ ] `denyPaths` includes `.env` and config files
- [ ] Workspace directory owned by `openclaw` user
- [ ] `.env` file is `chmod 600`
- [ ] `.env` file is owned by `openclaw:openclaw`
- [ ] No symlinks in workspace that point outside workspace

## Slack Security

- [ ] Socket Mode enabled (no public webhook URL)
- [ ] `dmPolicy` is `allowlist`
- [ ] `dmAllowlist` contains only authorized Slack user IDs
- [ ] Slack bot token has minimal required scopes
- [ ] Slack app token has only `connections:write` scope
- [ ] Token rotation is enabled in Slack app settings
- [ ] `streaming` is `false` (more stable, prevents partial message leaks)

## Network Security

- [ ] UFW firewall enabled
- [ ] Default incoming policy: DENY
- [ ] Only ports 22, 80, 443 open externally
- [ ] Port 3578 (gateway) closed externally
- [ ] Port 5678 (n8n) closed externally
- [ ] SSH key-only authentication (password auth disabled)
- [ ] SSH on non-standard port (optional but recommended)
- [ ] Fail2ban installed and configured

## Secrets Management

- [ ] All secrets in `.env` file, never in `openclaw.json`
- [ ] `.env` references used in config (`$VARIABLE_NAME` syntax)
- [ ] No secrets in identity stack files (SOUL.md, AGENTS.md, etc.)
- [ ] No secrets in logs — PII masking enabled
- [ ] Git repo does not contain `.env` (check `.gitignore`)
- [ ] API keys are scoped to minimum required permissions

## Token Rotation

- [ ] Gateway token rotated monthly
- [ ] Slack bot token rotated quarterly
- [ ] Slack app token rotated quarterly
- [ ] Anthropic API key rotated quarterly
- [ ] OpenRouter API key rotated quarterly
- [ ] n8n MCP token rotated monthly
- [ ] Rotation dates tracked in ops calendar

## systemd Hardening

- [ ] `NoNewPrivileges=true`
- [ ] `ProtectSystem=strict`
- [ ] `ProtectHome=true`
- [ ] `PrivateTmp=true`
- [ ] `ReadWritePaths` limited to workspace only
- [ ] `ReadOnlyPaths` for app and config
- [ ] `ProtectKernelTunables=true`
- [ ] `ProtectKernelModules=true`
- [ ] `ProtectControlGroups=true`
- [ ] `RestrictSUIDSGID=true`
- [ ] `RestrictNamespaces=true`
- [ ] `MemoryMax` set (prevent OOM)
- [ ] `CPUQuota` set (prevent runaway processes)
- [ ] `TasksMax` set (prevent fork bombs)

## Nginx Security (if exposed externally)

- [ ] HTTPS only — HTTP redirects to HTTPS
- [ ] SSL certificate valid and auto-renewing (Let's Encrypt)
- [ ] Security headers set (X-Frame-Options, CSP, etc.)
- [ ] Rate limiting configured
- [ ] Access to dotfiles denied
- [ ] n8n dashboard IP-restricted to office/VPN

## Monitoring & Audit

- [ ] Heartbeat checks running on schedule
- [ ] Alerts posting to dedicated Slack channel
- [ ] All agent tool calls logged
- [ ] Log retention policy defined (min 90 days for legal)
- [ ] Disk usage monitored
- [ ] Memory usage monitored
- [ ] Failed auth attempts logged and alerted

## Data Protection

- [ ] No client PII stored in agent memory — use system IDs only
- [ ] Conversation compaction enabled to limit data retention
- [ ] Workspace files reviewed for sensitive data monthly
- [ ] Backup encryption enabled for workspace backups
- [ ] Agent cannot access files outside workspace boundary
- [ ] Identity stack files do not contain client information

## Incident Response

- [ ] Escalation contacts defined (admin Slack IDs)
- [ ] Auto-recovery procedures documented
- [ ] Kill switch known: `sudo systemctl stop openclaw`
- [ ] Token revocation procedure documented for each provider
- [ ] Post-incident review process defined

---

## Review Schedule

| Check | Frequency | Responsible |
|---|---|---|
| Full checklist review | Monthly | Admin |
| Token rotation | Per schedule above | Admin |
| DM allowlist review | Monthly (or on staff change) | Admin |
| Log review | Weekly | Admin |
| Dependency updates | Monthly | Admin |
| Penetration testing | Annually | External vendor |

---

*Last reviewed: [DATE]*
*Reviewed by: [NAME]*

## Monthly Maintenance
- [ ] Rotate gateway token
- [ ] Review session logs for anomalies
- [ ] Update OpenClaw to latest version
