# Popy Agent Skills

Public agent skills by [Popy](https://popy.app) / Motivolog — install via
[skills.popy.app](https://skills.popy.app).

| Skill | What it does |
|---|---|
| [`game-dev`](game-dev/) | Accessibility-first kids' game development process for Flutter + Flame — from design doc to App Store, distilled from building [Sweet Pasta](https://popy.app/apps/sweet-pasta). |
| [`blind-playable-games`](blind-playable-games/) | Standalone add-on layer that makes any game project blind-playable — two audio channels, self-voicing onboarding, a global narration gesture, icon rules, eyes-closed verification. Composes with `game-dev`, which works fully without it. |

## Install

```sh
curl -fsSL https://skills.popy.app/i/popy-game-dev | sh
curl -fsSL https://skills.popy.app/i/popy-blind-playable-games | sh
```

or copy the skill folder (e.g. `game-dev/`) into your agent's skills
directory (`~/.claude/skills/` for Claude Code).

## License

MIT © Motivolog
