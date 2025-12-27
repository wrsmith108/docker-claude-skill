# Docker Development Skill for Claude Code

A Claude Code skill for container-based development with Docker.

## Installation

```bash
git clone https://github.com/wrsmith108/docker-claude-skill.git ~/.claude/skills/docker
```

## What's Included

- **Enforcement Rules**: Block npm/node commands from running on host
- **Pre-Flight Checks**: Verify container status before commands
- **Command Reference**: docker exec, docker-compose patterns
- **Troubleshooting**: Common issues and fixes
- **Sample docker-compose.yml**: Ready-to-use template

## Key Principle

**Never run `npm`, `node`, `npx`, `tsx`, or project scripts directly on the host machine.**

Always use `docker exec <container-name> <command>` instead.

## License

MIT - See [LICENSE](LICENSE)
