# Skills

All skills in this directory follow the [Agent Skills specification](https://agentskills.io/specification).

## What is an Agent Skill?

An Agent Skill is a portable, structured set of instructions that any compatible AI agent can discover and invoke. Each skill lives in its own directory with a `SKILL.md` file as the entry point.

## Structure

Every skill follows this layout:

```
skill-name/
├── SKILL.md          # Required — frontmatter + instructions
├── references/       # Optional — detailed reference docs loaded on demand
├── scripts/          # Optional — helper scripts
└── assets/           # Optional — templates, schemas, data files
```

`SKILL.md` frontmatter fields:

| Field | Required | Description |
|---|---|---|
| `name` | ✅ | Kebab-case, matches directory name |
| `description` | ✅ | What the skill does and when to use it |
| `compatibility` | optional | Environment or tool requirements |
| `license` | optional | License identifier |
| `metadata` | optional | Additional key/value pairs (author, version, etc.) |
| `allowed-tools` | optional | Pre-approved tools for the agent to run |

## Available Skills

| Skill | Description |
|---|---|
| [generate-playwright-test](./generate-playwright-test/SKILL.md) | Generate Playwright TypeScript e2e test files from BDD scenarios, test cases, or free-form descriptions |

## How to use

Any agent that supports the Agent Skills spec can load these skills automatically. See the [integration guide](https://agentskills.io/integrate-skills) for how to wire them into your agent or tool.

To validate a skill locally:

```sh
skills-ref validate ./generate-playwright-test
```

## Specification

- [agentskills.io/specification](https://agentskills.io/specification)
- [agentskills.io/integrate-skills](https://agentskills.io/integrate-skills)
