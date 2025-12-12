---
name: skill-finder
description: Search for and install agent skills from Hugging Face. Use this skill when users want to discover new skills, browse available skills on the Hub, or install skills to extend agent capabilities.
---

# Skill Finder: Discover and Install Agent Skills from Hugging Face

## Overview

Agent skills are packaged instructions and resources hosted on Hugging Face Spaces with the `agent-skill` tag. Use the Hugging Face MCP tools to search and install them.

## Searching for Skills

Use the `hf_space_search` MCP tool to find skills:

```python
# Search for all skills
hf_space_search(filter="agent-skill")

# Search with keywords
hf_space_search(filter="agent-skill", search="training")
```

## Installing Skills

Use the `hf_download` MCP tool to download a skill:

```python
# Download a skill space
hf_download(repo_id="hf-skills/llm-trainer", repo_type="space", local_dir=".claude/skills/llm-trainer")
```

## Skill Structure

A valid skill contains a `SKILL.md` file with YAML front matter:

```markdown
---
name: my-skill-name
description: When to use this skill and what it does.
---

# Instructions for the agent...
```

Optional directories: `scripts/`, `references/`, `templates/`
