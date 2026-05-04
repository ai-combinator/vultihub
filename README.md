# vultihub

Umbrella repo for the Vultisig agent stack. Adjacent clones of every sub-repo live next to this one; shared context (CLAUDE.md), docs, slide deck, and the ticket system live here at the root.

## Sub-repos

| Dir | What |
|---|---|
| `vultisig-sdk/` | Unified TypeScript SDK — all chain logic and signing |
| `agent-backend/` | Go AI chat service — Claude integration, MCP tool calls |
| `vultiagent-app/` | React Native mobile app (Expo) — chat-based wallet |
| `mcp-ts/` | MCP server — tool discovery, skills, chain interactions |

Each sub-repo has its own `CLAUDE.md`. The root `CLAUDE.md` applies across all of them.

## Tickets — source of truth

**All engineering tickets for the Vultisig stack live in this repo**, regardless of which sub-repo they target. This is true for both lightweight tickets and GitHub issues:

- **`.tickets/`** — source of truth for `tk` tickets (markdown + YAML frontmatter, managed via the `tk` CLI). Use for internal planning, specs, surgical work, design notes.
- **GitHub Issues on `ai-combinator/vultihub`** — source of truth for issues tracked on GitHub. Even if an issue concerns only `agent-backend` or `mcp-ts`, open it here.

Do NOT open issues or tickets on the sub-repos themselves. Cross-repo work would fragment across multiple issue trackers and lose traceability.

PRs still live on the sub-repo they target — a PR in `vultisig/agent-backend` that addresses ticket `v-qjvi` references the ticket ID in its body. Tickets can track dependencies across multiple PRs in different sub-repos.

## tk quick reference

```
tk ls                          # list open tickets
tk show <id>                   # show a ticket
tk new "<title>" -t <type>     # create a ticket (bug|feature|task|epic|chore)
tk dep <id> <dependency-id>    # add a dependency
tk close <id>                  # close a ticket
tk note <id> "<msg>"           # append a timestamped note
```

Ticket files are plain markdown — grep/edit them directly if `tk` is inconvenient.
