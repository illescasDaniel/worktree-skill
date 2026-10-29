# worktree-skill

An agent skill for the whole life of an isolated git worktree: create it, **set up its environment automatically**
(virtualenv, `node_modules`, build output), apply it back into its parent branch behind the project's quality gate,
save it, and delete it. A fresh worktree contains only tracked files, so without setup the first test run fails; the skill tells
the agent to do that setup on its own, following whatever the project documents in `AGENTS.md`, `CLAUDE.md` or the
README (often a language-specific script), and to fall back to copying or reinstalling from the lockfile.

It is an [Agent Skill](https://agentskills.io) (a single `SKILL.md`), so it works in Claude Code, Cursor and any other
agent that loads skills.

## How it works

| Command | What the agent does |
|---|---|
| `/worktree create` | `git worktree add -b` from the current branch, records that branch as the parent, then sets up the environment without being asked |
| `/worktree save` | Commits the task's files on the worktree branch; pushes only when asked |
| `/worktree apply` | Saves pending work, merges into the recorded parent branch (never a hard-coded main), resolves conflicts, runs the quality gate, then offers to delete |
| `/worktree delete` | Moves out, removes the worktree and its generated branch, tolerates editor file locks |

To make setup reliable, document it once in your project, for example a "Worktree dependencies" section in `AGENTS.md`
naming the script or commands to run. The skill looks there first.

## Install

### Any agent: the `skills` CLI

```bash
npx skills add illescasDaniel/worktree-skill
```

It asks which agents to install for and whether to install into the current project or globally. Non-interactive:
`npx skills add illescasDaniel/worktree-skill -a claude-code -y`, plus `-g` for global.

### By hand

Copy the `skills/worktree` folder into your agent's skills directory.

| | Project (shared with your team) | All your projects |
|---|---|---|
| Claude Code | `.claude/skills/worktree/` | `~/.claude/skills/worktree/` |
| Cursor | `.cursor/skills/worktree/` | `~/.cursor/skills/worktree/` |

## Contributing

Issues and pull requests are welcome. The skill is deliberately short: if a change makes `SKILL.md` longer, it should
earn it.

## License

MIT, see [LICENSE](LICENSE).
