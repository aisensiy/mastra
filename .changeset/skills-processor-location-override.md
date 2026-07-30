---
'@mastra/core': patch
---

SkillsProcessor: the `location` field injected into the system prompt is a path on the server running the agent (`${skill.path}/SKILL.md`). When the agent's filesystem tools operate elsewhere — e.g. a sandbox workspace — that path is meaningless to the model, and the accompanying instruction ("use the skill path shown in the location field") steers it into calling filesystem tools with a path that does not exist there.

Two changes:

- New `formatLocation` option on `SkillsProcessorOptions` lets embedders control how the location is rendered (e.g. remap it to where skill files are actually mounted in the sandbox, or emit a plain identifier).

```ts
new SkillsProcessor({
  workspace,
  formatLocation: skill => `/opt/skills/${skill.name}/SKILL.md`,
});
```

- The fixed skill-tool instruction now clarifies that the location identifies a skill for the `skill`/`skill_read` tools and is not guaranteed to exist on the workspace filesystem, so skill files should be read with `skill_read` rather than filesystem tools.
