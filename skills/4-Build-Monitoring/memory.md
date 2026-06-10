# Memory: /add-monitoring

Write an entry when the skill assumed something that wasn't true for this run.
Max 7 entries. Newest first. One line each.
If a pattern appears twice → add it to SKILL.md, then delete it here.

- User needs to know WHAT is being monitored (specific to their app) before setup begins — generate the "Here's what we'll monitor" summary from pattern-finder results before asking for any accounts or credentials.
- Users don't know where to paste credentials — always say "Paste directly in this chat" in the same message as the credential request.
- Dev server restart after env vars is not obvious to non-developers — always state it explicitly before asking them to verify.
