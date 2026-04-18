Hey 👋 @zscole

I ran your skills through `tessl skill review` at work and found some targeted improvements. Here's the full before/after:

| Skill | Before | After | Change |
|-------|--------|-------|--------|
| bagman (SKILL.md) | 10% | 88% | +78% |
| bagman (openclaw) | 80% | 80% | — |

![Score Card](score_card.png)

The root `SKILL.md` was missing YAML frontmatter entirely, which meant it failed validation and scored 10%. The openclaw variant already had solid frontmatter and scored 80% — no changes needed there.

<details>
<summary>Changes made</summary>

- Added YAML frontmatter with `name`, `description`, and explicit "Use when..." trigger keywords
- Condensed the architecture ASCII diagram into a compact tree format
- Replaced the verbose "Patterns Detected" table with a single-line summary
- Trimmed the operation allowlisting code block to show just the essential pattern
- Replaced the full confirmation flow code with a pointer to the implementation file
- Condensed the defense layers ASCII diagram into a single pipeline line
- Condensed the "Common Mistakes" section from code blocks to a bullet list
- Trimmed the "Security Model Limitations" section to a single line

</details>

I kept this PR focused on the root `SKILL.md` which had the biggest improvement to keep the diff reviewable. The openclaw variant is already well-structured at 80%. Happy to follow up with further improvements in a separate PR if you'd like.

Honest disclosure — I work at @tesslio where we build tooling around skills like these. Not a pitch - just saw room for improvement and wanted to contribute.

Want to self-improve your skills? Just point your agent (Claude Code, Codex, etc.) at [this Tessl guide](https://docs.tessl.io/evaluate/optimize-a-skill-using-best-practices) and ask it to optimize your skill. Ping me - [@yogesh-tessl](https://github.com/yogesh-tessl) - if you hit any snags.

Thanks in advance 🙏
