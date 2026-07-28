# robertford-claude-skills

A small personal marketplace of Claude Code / Claude Cowork plugins.

## Included plugins

### `substack-seo`

Audits, writes, and applies SEO titles + meta descriptions across every post
on a Substack publication. It reads each post's real text before writing
copy, applies changes live through the post editor, and verifies everything
persisted via Substack's own API. Built for use inside Claude Cowork with
browser automation (Claude in Chrome), but degrades gracefully to a
worksheet-only mode if no browser is available.

See plugins/substack-seo/skills/substack-seo/SKILL.md
for the full workflow this skill follows.

## Installing

If you use Claude Code:

/plugin marketplace add fordrm/substack-seo-marketplace
/plugin install substack-seo@robertford-claude-skills

If you use Claude Cowork, add this same repo as a marketplace from Cowork's
plugin settings, then install substack-seo from it.

After installing, run /reload-plugins if you added it mid-session.

## Updating

/plugin marketplace update robertford-claude-skills

## License

Feel free to fork, modify, and redistribute.
