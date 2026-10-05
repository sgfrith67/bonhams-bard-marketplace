# Bonhams BARD plugin marketplace

Private marketplace for the `bonhams-bard` plugin (BARD, the Bonhams Auction Research Database).

## Install (Claude Code)

```
/plugin marketplace add sgfrith67/bonhams-bard-marketplace
/plugin install bonhams-bard@bonhams-bard-marketplace
```

You need read access to this repository. On first use, sign in to the `bonhams-bard` connector when prompted.

## Updating

Change files under `plugins/bonhams-bard/`, bump `version` in both `plugins/bonhams-bard/.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`, then commit and push. Users get the update with `/plugin marketplace update bonhams-bard-marketplace`.
