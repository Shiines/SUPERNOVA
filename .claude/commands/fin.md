# /fin — Session Close

## What It Does
Closes the session cleanly: updates memory, writes meeting notes, pushes to Git.

## Steps
1. Update `primer.md` — done / next / blockers / finances
2. Write `memory/reunions/CR-[date].md` — summary of session
3. Update `00-system/GENESIS.md` if there's a system learning
4. Run: `git add -A && git commit -m "Session [date]" && git push`
5. Run `/clear`

## Output
Confirmation:
```
/fin complete — [date]
primer.md updated
CR saved → memory/reunions/CR-[date].md
Git pushed
Context cleared — see you tomorrow.
```
