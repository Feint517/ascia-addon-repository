# The ASCIA add-on version ledger

Every add-on version ever handed out, one file per add-on (`addons/<addon>.json`): its state
(`reserved`, then `built`), the source commit it was built from, and each architecture's image
digest. Written by `uv run ascia ledger` / `ascia image build` in ascia-addons
(docs/X86_BUILD_PLAN.md, D6 and D10), never by hand.

A version is reserved by a push before it is built — a rejected push is the lock — so no number is
ever handed out twice, and a failed build burns its number rather than reusing it. Supervisor
compares version strings and never re-pulls one it has, so a reused number would be an appliance
that silently keeps the old image.

**Nothing on an appliance or the HAOS VM reads this branch.** The channels they read are `edge` and
`main`.
