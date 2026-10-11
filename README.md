# AJ AI Stuff

Personal Claude Code material, kept outside any project repo.

## Layout

| Folder                               | What goes there                                                                          | How it is used                           |
|--------------------------------------|------------------------------------------------------------------------------------------|------------------------------------------|
| `skills/<name>/SKILL.md`             | Instructions Claude loads on its own (needs `name` + `description` frontmatter)          | Link into `~/.claude/skills/`            |
| `guides/<topic>.md`                  | Human how-tos and reminders (setup steps, workflows). Not loaded by Claude automatically | Read, or point Claude at it with `@path` |
| `rules/<topic>.md` (add when needed) | Short always-on rules or conventions to copy into a project's `CLAUDE.md`                | Copy or `@`-import                       |

## Install the skills (personal, all projects)

```bash
for d in ~/code/aj-ai-stuff/skills/*/; do
  ln -sfn "$d" ~/.claude/skills/$(basename "$d")
done
```

## Guide template

Each guide starts with: purpose (one line), prerequisites, steps, and "when not to use".
