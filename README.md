# whotag-skills

Claude Skills that pair with the [whotag](https://whotag.ai) MCP connector.

Each skill encodes campaign-workflow judgment (when to paginate vs. re-query,
when to reveal contacts, how to segment results into multiple buckets) that
tool descriptions alone cannot convey.

## Skills

| Name | Description |
| --- | --- |
| [`whotag-influencer`](./whotag-influencer/) | End-to-end workflow guide for the whotag connector — search influencers, extract emails/contacts, organize them into buckets, and export to Excel. |

## Installation

Skills in this repository are intended to be loaded by Claude clients
(Claude.ai, Claude Desktop, Claude Code) alongside the whotag MCP connector.

To use a skill locally with Claude Code, copy the skill directory into your
project's `.claude/skills/` folder:

```bash
git clone https://github.com/vaiv-danlee/whotag-skills.git
cp -r whotag-skills/whotag-influencer /path/to/your/project/.claude/skills/
```

## Related

- whotag MCP connector — natural-language influencer search, contact reveal,
  and bucket management tools that these skills orchestrate.

## License

MIT
