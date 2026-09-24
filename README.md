# ajn-skills

Claude Code skills, packaged as a plugin.

| Skill | What it does |
|---|---|
| `explain-diff-html` | Explains a diff, branch, or pull request as an interactive page published as a Claude Artifact. |
| `recommit-branch` | Rewrites a finished branch into one commit per idea, keeping the final tree identical. |

## Install

In Claude Code, run:

```
/plugin marketplace add kinnou02/ajn-skills
/plugin install ajn-skills@ajn-skills
```

Skills are then available as `ajn-skills:<skill>`, for example `/ajn-skills:recommit-branch`.

To get updates, run `/plugin marketplace update ajn-skills`.
