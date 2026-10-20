---
name: email-templates
description: Email template workflows for common correspondence patterns
version: 1.0.0
author: James Watt
license: MIT
platforms: [macos, linux]
metadata:
  hermes:
    tags: [email, correspondence, automation]
    category: productivity
---

# Email Templates Skill

## When to Use
- Sending common email types (meeting requests, follow-ups, status updates)
- Standardizing email format across teams
- Automating email composition with templates

## Procedure

### Template Structure
```markdown
# Subject: [Topic] - [Brief description]

Hi [Name],

[Opening sentence]

[Main message with details]

[Call to action or next steps]

Best regards,
[Your Name]
```

### Using Templates with Hermes
```bash
# Load and customize a template
hermes chat --toolsets skills -q "Use the email-templates skill to send a meeting request to john@example.com about project kickoff on Friday at 2pm"

# Or manually fill template variables
template="# Subject: Project Update

Hi {{name}},

Here's the latest on the project: {{update}}

Next steps: {{steps}}

Best,
{{sender}}"
```

### Common Template Patterns

**Meeting Request**
```
Subject: Meeting: [Topic] on [Date] at [Time]

Hi [Name],

I'd like to schedule a meeting to discuss [Topic].

Proposed times:
- [Date] at [Time]
- [Date] at [Time]
- [Date] at [Time]

Please let me know what works best.

Best regards,
[Your Name]
```

**Follow-up Email**
```
Subject: Follow-up: [Previous Topic]

Hi [Name],

Following up on my previous message regarding [Topic].

Any updates on [Topic]?

Looking forward to hearing from you.

Best regards,
[Your Name]
```

**Status Update**
```
Subject: Weekly Status Update: [Week of Date]

Hi Team,

[High-level summary of accomplishments]

[Blockers or challenges]

[Plans for next week]

Questions or feedback welcome.

Best regards,
[Your Name]
```

## Pitfalls
- **Overly long templates** — keep them scannable; use sections/headers
- **Missing variable placeholders** — clearly mark what needs substitution (e.g., `{{name}}`)
- **Forgetting sign-off** — always include name and contact info

## Verification
```bash
# Test template generation
template="# Subject: Test

Hi [Name],

This is a test email.

Best,
[Sender]"

echo "$template" | head -5
```
Expected: renders first 3 lines correctly with placeholder text.

# Common Templates Reference
| Template | Purpose |
|---|---|
| Meeting Request | Schedule meetings with proposed times |
| Follow-up | Follow up on pending items |
| Status Update | Weekly progress reports |
| Project Kickoff | New project initiation |
| Client Onboarding | New client welcome sequence |