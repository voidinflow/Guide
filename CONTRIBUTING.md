# Contributing

This guide is meant to stay **practical** as tools and best practices change. Contributions that improve clarity, correctness, and repeatability are highly valued.

## What makes a great contribution

- **Specific**: “Change X to Y because Z” beats “this is confusing”
- **Reproducible**: steps that a reader can follow without tribal knowledge
- **Opinionated but justified**: if you recommend a pattern, say why and when it breaks
- **Small surface area**: prefer tight edits over sweeping rewrites (unless you’re fixing structure)

## Fast paths (pick one)

### Small (5–15 minutes)

- Fix typos, broken links, misleading phrasing
- Add a missing “gotcha” or a short checklist item
- Improve a prompt template (shorter, more enforceable constraints)

### Medium (30–90 minutes)

- Add a new section with a concrete template + example
- Update a tool configuration path / key that changed
- Add a “failure mode → diagnosis → fix” block to any chapter

### Large (half day+)

- New chapter or major restructure (open an issue first so we can align)
- Full end-to-end workflow example
- Security review improvements (threat model + test prompts + hardening checklist)

## Style conventions

These conventions make the docs easier to read and contribute to:

- **Use `##` / `###` headings** (avoid `#` except the title)
- **Prefer short paragraphs + bullets** over walls of text
- **Be explicit about scope** (project-wide vs folder-specific vs per-task)
- **Use code blocks for copy/paste**, but keep them minimal and durable
- **Avoid vendor marketing language**; call out trade-offs and failure modes

## Navigation conventions

- Each chapter should have a top navigation line like:
  - `← [Back to Index](./README.md) | [Next →](./next.md)`
- If you add a new file, link it from `README.md`’s “Playbook Structure” table.

## PR checklist

- [ ] The change is **accurate** (no invented file paths, settings keys, or tool behavior)
- [ ] The change is **scoped** (doesn’t rewrite unrelated parts)
- [ ] Links work (relative links preferred)
- [ ] Templates are copy/pasteable and **minimal**
- [ ] If you added a new tool/setup instruction, include at least one **gotcha**

## Good first contributions

If you want to help but don’t know where to start:

- Add “common mistakes” rows to chapters that don’t have them yet
- Add a short “test plan” pattern to templates that generate code
- Update fast-moving editor/tool setup steps in [`tools.md`](./tools.md)

