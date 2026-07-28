# substack-seo

A Claude Skill that turns "improve my Substack SEO" into a repeatable,
verified workflow: discover, research, draft, apply, verify.

- Discover every published post via Substack's public archive API and
  flag which ones are missing custom SEO.
- Research by reading each post's actual text, never guessing SEO copy
  from the stylized on-page title alone.
- Draft a title (60 chars or fewer) and description (roughly 140-160 chars)
  grounded in what the post actually says, matched to the author's voice.
- Apply the fields live through the post editor (requires a connected
  browser, e.g. Claude in Chrome in Cowork).
- Verify every change actually persisted via a read-only API check,
  never trusting a Save click alone.

Built from direct experience applying SEO to 43 live posts across two
Substack publications, including documenting the dead ends (like a rejected
direct API write) so they are not relearned.

See skills/substack-seo/SKILL.md for the complete instructions Claude
follows when this skill is invoked.
