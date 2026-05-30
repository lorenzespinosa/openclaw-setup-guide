# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [3.1.0] - 2026-02-15

### Added
- FAQ section
- Troubleshooting table
- Monthly maintenance checklist

---

## [3.0.0] - 2025-10-15

### Added
- n8n MCP integration guide with full architecture diagrams
- MCP tool documentation in TOOLS.md example
- n8n systemd service file
- MCP configuration section in openclaw.json example
- Cost optimization section with tiered model routing
- Monthly cost estimate table

### Changed
- Updated architecture diagrams to include MCP flow
- Expanded security checklist with MCP-specific items
- Updated agent configs to include MCP-aware ops-agent

---

## [2.0.0] - 2025-08-20

### Added
- Multi-agent architecture guide (orchestrator + specialists)
- AGENTS.md example with routing rules and decision tree
- USER.md example with role-based access matrix
- HEARTBEAT.md example with health monitoring config
- Agent routing Mermaid diagrams
- Specialist agent configs (ops, research, drafting)
- Session spawning documentation
- Slack DM allowlist behavior table

### Changed
- Restructured openclaw.json example for multi-agent setup
- Expanded security section with per-agent tool control
- Updated systemd service with tighter sandboxing

### Fixed
- Corrected nginx WebSocket proxy headers
- Fixed systemd ReadWritePaths scope

---

## [1.0.0] - 2025-06-10

### Added
- Initial setup guide for OpenClaw on Ubuntu with systemd
- Single-agent configuration with identity stack
- SOUL.md example for law firm persona
- TOOLS.md example with built-in tool reference
- openclaw.json example with gateway and Slack config
- systemd unit file for OpenClaw
- nginx reverse proxy config with WebSocket support
- Security hardening checklist
- Slack bot setup walkthrough (Socket Mode, app manifest)
- Installation guide (Node.js, service user, environment vars)
- MIT License
- Contributing guidelines
- Troubleshooting section
