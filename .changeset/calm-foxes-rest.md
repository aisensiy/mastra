---
'@mastra/core': patch
---

Reduced how much a durable agent writes for each run snapshot, again.

Every completed step of a durable run kept a copy of the conversation, the accumulated step records and the previous step result in the input it had been called with, and all of that was rewritten on every later save. Completed steps are never run again, so those copies are now left out. A five step run in our benchmark went from 23.7 MB to 16.2 MB persisted, and the saving grows with conversation length. Suspended steps, step outputs and resume behavior are unchanged.
