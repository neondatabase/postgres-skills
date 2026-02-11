# Postgres Skills

A collection of [Agent Skills](https://agentskills.io/) for Postgres best practices.

## What are Agent Skills?

As per the Agent Skills spec, skills are folders of instructions, scripts, and resources that agents can discover and use to do things more accurately and efficiently. Once installed, skills are automatically invoked by the agent upon detection of relevant tasks.

It all starts with the `SKILL.md` file in the skill's directory. It's the entry point and allows agents to progressively discover information as needed.

## Available Skills

### Postgres Best Practices

[skills/postgres-best-practices](skills/postgres-best-practices/SKILL.md)

Best practices and guidelines for working with Postgres. Covers schema design, indexing, query optimization, and common pitfalls.

## Installation

```bash
npx skills add neondatabase/postgres-skills
```
