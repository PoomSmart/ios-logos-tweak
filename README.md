# ios-logos-tweak

Agent skill: best practices for iOS jailbreak tweaks built with [Theos](https://theos.dev) and Logos. The instructions apply to any coding assistant that can load `SKILL.md`.

## Contents

| File | Purpose |
|------|---------|
| [SKILL.md](SKILL.md) | Main instructions (Logos, Substrate, messaging, ivars) |
| [reference.md](reference.md) | Logos vs manual `MSHook*` reference |

## Usage

Use when writing or reviewing tweak code: **`Tweak.x`** by default, `.xm` when needed, hooks, swizzling, Theos projects.

The skill name in frontmatter is **`ios-logos-tweak`**. Keep **`reference.md`** next to **`SKILL.md`** so relative links work.

| Tool | How to invoke (typical) |
|------|-------------------------|
| [Cursor](https://cursor.com) | Attach the skill in chat, or `@ios-logos-tweak` if your setup exposes it |
| [Claude Code](https://code.claude.com) | `/ios-logos-tweak`, or let Claude match from the `description` |
| [Codex](https://developers.openai.com/codex) (CLI / IDE / ChatGPT desktop) | `$ios-logos-tweak`, `/skills`, or `@` skill picker |

Other assistants: paste or import `SKILL.md`, or add the repo to whatever skills/rules path they support.

## Install

Clone the **whole repository** (not only `SKILL.md`) into the skill folder for your tool. Replace `<repo-url>` with your Git remote.

### Personal (all projects on this machine)

```bash
git clone <repo-url> ~/.cursor/skills/ios-logos-tweak      # Cursor
git clone <repo-url> ~/.claude/skills/ios-logos-tweak      # Claude Code
git clone <repo-url> ~/.agents/skills/ios-logos-tweak      # Codex / Agent Skills layout
```

Pick one path per tool you use. To avoid three clones, clone once and symlink:

```bash
git clone <repo-url> ~/src/ios-logos-tweak
ln -s ~/src/ios-logos-tweak ~/.cursor/skills/ios-logos-tweak
ln -s ~/src/ios-logos-tweak ~/.claude/skills/ios-logos-tweak
ln -s ~/src/ios-logos-tweak ~/.agents/skills/ios-logos-tweak
```

Create parent directories if needed (`mkdir -p ~/.cursor/skills` etc.).

### Project (team / one repo)

```bash
git clone <repo-url> .cursor/skills/ios-logos-tweak    # Cursor
git clone <repo-url> .claude/skills/ios-logos-tweak    # Claude Code
git clone <repo-url> .agents/skills/ios-logos-tweak    # Codex
```

Or add as a git submodule at the same paths. Commit the skill directory so teammates get it.

### Notes

- **Claude Code:** if `~/.claude/skills` did not exist when the session started, run `/reload-skills`.
- **Cursor:** user skills live under `~/.cursor/skills/`; project skills under `.cursor/skills/`.
- **Editors without a skills folder** (e.g. Copilot `copilot-instructions.md`, generic `AGENTS.md`): copy the body of `SKILL.md` (below the frontmatter) into your project rules, or link to this repo in those files and ask the agent to read `SKILL.md` when working on tweaks.

## License

[MIT](LICENSE)
