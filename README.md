# PumaCC Skills

Reusable Claude Code skills for PumaCC agents. Each skill is a self-contained capability pack that can be installed into any Claude agent workspace.

## Structure

```
skills/
  marketplace.json          # Marketplace catalog — list of all published skills
  <skill-slug>/
    SKILL.md                # Skill definition (frontmatter: name, slug, version, description)
    _meta.json              # Publisher metadata (ownerId, publishedAt)
    *.md                    # Supporting reference docs
```

## Installing a Skill

Copy the skill directory into your agent's `.openclaw/skills/`:

```sh
cp -r <skill-slug> /path/to/agent/.openclaw/skills/
```

## Adding a Skill

1. Create a new directory named after the skill slug
2. Add `SKILL.md` with YAML frontmatter (`name`, `slug`, `version`, `description`)
3. Add `_meta.json` with publisher metadata
4. Add reference `.md` files as needed
5. Register the skill in `marketplace.json`

## Skills

| Slug | Name | Description |
|------|------|-------------|
| `api` | API Conventions | REST API integration patterns — auth, retries, pagination, webhooks |
| `frontend` | Frontend Design | React/Next.js/Tailwind UI development |
| `github` | GitHub | `gh` CLI usage for issues, PRs, CI |
