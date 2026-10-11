# Spec-Driven Development (SDD) with OpenSpec

OpenSpec commands (`/opsx:*`) are typed into the AI assistant's chat (Claude Code, Cursor), **not** into a standard terminal.

## Setup (one time)

### Prerequisites
- Node.js 20.19.0 or newer (`node -v`)
- An AI assistant that supports slash commands (Claude Code, Cursor)

### 1. Install the OpenSpec CLI

```bash
npm install -g @fission-ai/openspec@latest
openspec --version   # check it works
```

### 2. Initialize OpenSpec in the project

Run from the repo root (`trust/`):

```bash
openspec init
```

- Select your AI tool(s) when prompted (Claude Code, and/or Cursor).
- This creates the `openspec/` directory (`specs/`, `changes/`) and generates the `/opsx:*` slash commands for the selected tool.
- Commit `openspec/` and the generated command files so the whole team shares them.

### 3. Restart the assistant

Restart Claude Code (or reload Cursor) so it picks up the new `/opsx:*` commands. Type `/opsx:` to confirm they appear.

### Updating later

```bash
npm install -g @fission-ai/openspec@latest
openspec update   # refreshes the generated commands/instructions in the project
```

> Package name and commands are from my knowledge of OpenSpec. If something differs, check `openspec --help` or the project README (github.com/Fission-AI/OpenSpec).

---

## Flow at a glance

| # | Step               | Command                    | Who acts | Output                                      |
|---|--------------------|----------------------------|----------|---------------------------------------------|
| 1 | Explore (optional) | `/opsx:explore`            | AI       | Understanding of the current state          |
| 2 | Propose            | `/opsx:propose "<change>"` | AI       | `openspec/changes/<change>/` planning files |
| 3 | Review and correct | —                          | **You**  | Approved specs and design                   |
| 4 | Apply              | `/opsx:apply`              | AI       | Code, following `tasks.md`                  |
| 5 | Verify             | `/opsx:verify`             | AI       | Gap report against scenarios                |
| 6 | Archive            | `/opsx:archive`            | AI       | Change archived, specs merged               |

---

## 1. Explore the current flow (optional, recommended for existing systems)

Have the AI map the existing behavior before you change it.

```
/opsx:explore
```

Ask it to, for example (KYC):
- analyze the current KYC code,
- identify where user states are updated,
- list existing third-party integrations.

## 2. Propose the change

```
/opsx:propose "Update the existing KYC verification process"
```

This creates a change directory, e.g. `openspec/changes/update-kyc/`, with:

| File          | Contents                                                 |
|---------------|----------------------------------------------------------|
| `proposal.md` | The problem, and what will be ADDED, MODIFIED or REMOVED |
| `specs/`      | Behavioral scenarios in GIVEN/WHEN/THEN form             |
| `design.md`   | Technical approach and architectural choices             |
| `tasks.md`    | Step-by-step implementation checklist in atomic chunks   |

Example scenario:
> GIVEN a user uploads an expired ID, WHEN verified, THEN transition state to `manual_review`.

## 3. Review and correct the blueprint (human step)

Open the generated files in your editor and read them. **The AI follows these documents strictly when coding**, so fix problems now.

Check for:
- missed edge cases (e.g. network timeout from the identity provider),
- wrong library or architectural choice,
- scenarios that don't match the intended behavior.

Fix them in `specs/` or `design.md`.

## 4. Apply the implementation

Once you approve the plan:

```
/opsx:apply
```

The AI works through `tasks.md` one task at a time, writing code that follows the specs.

Progress is tracked in the files, so if the session gets too long you can clear the chat context and resume where you left off.

## 5. Verify

```
/opsx:verify
```

Checks the implementation against the GIVEN/WHEN/THEN scenarios and reports unaddressed gaps. Fix the gaps and re-run until everything passes.

## 6. Archive

```
/opsx:archive
```

Moves the change into the archive and merges the new specs into the canonical `openspec/specs/` directory. That directory is the system's living documentation.
