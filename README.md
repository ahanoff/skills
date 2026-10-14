# Agent Skills

[Agent Skills](https://agentskills.io) for infrastructure work with coding agents: Claude Code, Codex, Cursor and anything else that loads `SKILL.md` files. Each skill is a runbook written down from a real production change, with the background on [ahanoff.dev](https://ahanoff.dev).

| Skill | What it does | Background |
| --- | --- | --- |
| [`rds-postgres-major-upgrade`](skills/rds-postgres-major-upgrade/SKILL.md) | Major version upgrade of RDS for PostgreSQL with a Blue/Green deployment driven through the RDS API, then Pulumi state reconciled, keeping the old instance as a rollback | [Upgrading RDS PostgreSQL with Blue/Green: what Pulumi's guide leaves out](https://ahanoff.dev/blog/rds-blue-green-with-pulumi-what-the-guide-leaves-out) |

## Install

### Claude Code

This repository is a plugin marketplace. Skills are grouped into plugins by area:

```bash
claude plugin marketplace add ahanoff/skills
claude plugin install aws-rds@ahanoff-skills
```

### Any agent

With the [`skills`](https://github.com/vercel-labs/skills) CLI:

```bash
npx skills add ahanoff/skills --skill rds-postgres-major-upgrade
```

Or copy a skill directory into your agent's skills directory, such as `~/.claude/skills/` or `.agents/skills/`.

## Layout

```text
skills/<skill-name>/SKILL.md      # one directory per skill, Agent Skills format
.claude-plugin/marketplace.json   # Claude Code marketplace: plugins and the skills each one loads
```

## Ground rules for every skill

- No account IDs, hostnames or credentials. Skills ask for the AWS profile and region and load secrets only inside the command that needs them.
- Read-only checks run freely. Every step that changes production waits for a person's explicit go.

## License

[Apache-2.0](LICENSE)
