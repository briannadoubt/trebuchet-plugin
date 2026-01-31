# CLAUDE.md

This file provides guidance to Claude Code when working with this plugin repository.

## Release Process

This project uses semantic versioning **WITHOUT** the 'v' prefix for tags.

### Creating a Release

```bash
# Tag format: MAJOR.MINOR.PATCH (no 'v' prefix)
git tag -a 1.3.0 -m "Release message here"
git push origin 1.3.0
```

### Version Pattern

- ✅ Use: `1.0.0`, `1.2.3`, `2.0.0`
- ❌ Don't use: `v1.0.0`, `v1.2.3`, `v2.0.0`

**IMPORTANT:** Always use semantic versioning **without** the 'v' prefix when creating tags.

## Plugin Structure

```
trebuchet-plugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   ├── SKILLS.md                # Skills index
│   ├── getting-started/
│   │   └── SKILL.md
│   ├── distributed-actors/
│   │   └── SKILL.md
│   └── ...
├── agents/
│   └── trebuchet-expert.md      # Specialized agent
├── CLAUDE.md                    # This file
├── README.md
└── LICENSE
```

## Updating the Plugin

When updating skills or agents:

1. Make changes to skill files
2. Test the changes locally
3. Commit with descriptive message
4. Tag a new release (without 'v' prefix)
5. Update the claude-marketplace repo with new version

## Syncing with Trebuchet Framework

When Trebuchet releases a new version:

1. Review CHANGELOG in Trebuchet repo
2. Update all relevant skills with new APIs
3. Update trebuchet-expert agent knowledge
4. Document any deprecated APIs
5. Tag a new plugin release
6. Update marketplace version reference
