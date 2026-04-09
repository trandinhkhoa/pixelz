---
trigger: always_on
---
# decision-protocol.md

Before making any design decision — including but not limited to:
- choosing a library or dependency
- structuring folders or modules
- defining data models or interfaces
- picking an architectural pattern
- creating abstractions or shared utilities

STOP and ask me. Present:
1. What decision needs to be made
2. 2-3 options with brief tradeoffs
3. Your recommendation and why

Do not proceed until I confirm.

Whenever a design decision is made (whether I initiated the question or you decided independently),
append an entry to `docs/DECISIONS.md` in this format:

## [short decision title] — {date}
**Context:** why this decision point came up
**Options considered:** what alternatives existed
**Decision:** what was chosen
**Reason:** why