# My Claude Code skills

Personal skills created with Claude Code. This repo lives at `~/.claude/skills/`.

| Skill | Use |
|---|---|
| [app-store-publish](app-store-publish/SKILL.md) | Publish an iOS app (Flutter / Xcode) on App Store Connect: signing, upload, TestFlight, listing, screenshots, privacy, review. |

## Install on a new Mac
```bash
git clone https://github.com/jeanbaptisteallaire/claude-skills.git ~/.claude/skills
```
(If `~/.claude/skills` already exists, clone elsewhere and copy the skill folders in.)

## Add a new skill
Create `~/.claude/skills/<name>/SKILL.md`, add `!/<name>/` to `.gitignore`, then commit and push.
