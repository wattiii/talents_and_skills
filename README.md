# talents_and_skills

This is where I will share public talents and skills.

## Available Skills

I've published 3 individual skills that can be installed selectively:

| Skill | Description | Install |
|---|---|---|
| `git-workflow` | Common git workflows and automation patterns | `hermes skills install wattiii/talents_and_skills/git-workflow` |
| `pdf-processor` | PDF processing workflows: extraction, transformation | `hermes skills install wattiii/talents_and_skills/pdf-processor` |
| `email-templates` | Email template workflows for common correspondence | `hermes skills install wattiii/talents_and_skills/email-templates` |

### How to Install

```bash
# Add the tap (one-time)
hermes skills tap add wattiii/talents_and_skills

# Install individual skills you need
hermes skills install wattiii/talents_and_skills/git-workflow
hermes skills install wattiii/talents_and_skills/pdf-processor
hermes skills install wattiii/talents_and_skills/email-templates
```

### Search & Browse

```bash
# Search for skills by keyword
hermes skills search git-workflow --source github

# List all skills from this tap
hermes skills list --source wattiii/talents_and_skills
```

Each skill is independently versioned and includes full SKILL.md documentation with procedures, pitfalls, and verification steps.
