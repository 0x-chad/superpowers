# Superpowers Development Notes

**After any commit or change:** Always use Method 1 (version bump) to refresh the plugin.

## Refreshing Plugin After Edits

Claude Code caches plugins to `~/.claude/plugins/cache/` even for local directory sources. Edits to source files don't auto-propagate.

**Method 1: Version bump (clean)**
```bash
# 1. Edit source files
# 2. Bump version in .claude-plugin/plugin.json
# 3. Update plugin
claude plugin update superpowers@superpowers-local
# 4. Restart claude
```

**Method 2: Manual cache clear (fast)**
```bash
rm -rf ~/.claude/plugins/cache/superpowers-local
# Restart claude
```

| Method | Best for |
|--------|----------|
| Version bump | Tracked releases, clean versioning |
| Manual clear | Rapid iteration, testing |
