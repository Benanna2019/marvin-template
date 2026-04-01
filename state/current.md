# Current State

Last updated: 2026-04-01

## Active Priorities

1. **DEV-276: Step metadata docs.** Two PRs open and ready for re-review after addressing Dan's feedback:
   - PR #1510 (reference): Now at correct path `/docs/reference/typescript/v4/functions/metadata`. Redirect from old features path in place.
   - PR #1511 (how-to): Cross-link updated to new reference URL.
   - Still waiting on Andy re: "userland" visibility in dashboard UI.
   - Screenshots, retry behavior callout, and "How it works" section still unaddressed.
2. **Sterling friction log tickets:** DEV-288, DEV-289, DEV-290 created but still in Triage. Need to be moved to backlog or started.
3. **Doc audit backlog:** INNGEST_DEV=1 callouts, serving-inngest-functions v4, self-hosting cleanup, auto integration section, v4 API fixes (rewriteGatewayEndpoint -> gatewayUrl).
4. **Codemod (DEV-290):** Sweep v3 patterns to v4 across docs and examples.

## Completed Recently

- PR #1495 (Next.js quickstart): Merged 2026-03-27.
- Step metadata reference page moved to correct path in reference section (2026-04-01).
- All of Dan's naming and framing feedback addressed on both metadata PRs.
- Sterling friction log reviewed, three Linear tickets created (2026-03-31).

## Open Threads

- Andy DM about "userland" display in dashboard (sent, awaiting response)
- Durable endpoint streaming: Linell posted demo in #dev-rel-docs (2026-03-26). Will eventually need docs.
- Docs Game Plan page in Notion updated 2026-03-28
- Diataxis skill built locally at .claude/skills/diataxis.md, not yet contributed to shared team repo

## Platform Notes

MARVIN now running in Cowork (claude.ai desktop app). Workspace folder mounted at marvin-template repo. Can read/write files, commit to git, and push to GitHub. Session state persists via commits to this repo.

## Recent Context

- 2026-04-01: Moved metadata reference page to correct URL path per Dan's review. Updated cross-links. Added redirect. Session context committed.
- 2026-03-31: Created Linear tickets from Sterling's friction log. Split step metadata into Reference + How-to pages. Opened PRs #1510 and #1511. Worked through Dan's review (naming, framing, path). Key insight: step.metadata is a step tool, not metadata about a step.
- 2026-03-30: Morning digest sent to Slack DM. Reviewed Boris Cherny's Claude Code features thread. Decided to move entirely off terminal to claude.ai/code.

---

*This file is the source of truth for session continuity. MARVIN reads it on startup and updates it on every /end and /update. If "Last updated" is more than 3 days old, MARVIN will suggest a full refresh.*
