# secrets-checker

**A Claude Code plugin that prevents secrets from being committed to git.**

## Overview

This plugin automatically scans staged files before git commits and blocks the commit if it detects potential secrets like API keys, tokens, passwords, or hardcoded values from `.env` files.

## Installation

```bash
# Add the local marketplace
/plugin marketplace add /home/jamesprial/code/claude-plugins

# Install the plugin
/plugin install secrets-checker@plugins-by-james
```

## How It Works

The plugin uses a **PreToolUse hook** that intercepts `git commit` commands:

1. Reads your `.env` file to identify secret values
2. Scans all staged files for:
   - Pattern-based secrets (AWS keys, API tokens, private keys, etc.)
   - Hardcoded values from your `.env` file
3. **Blocks the commit** if secrets are detected
4. Shows detailed warnings with file paths and line numbers

## Detected Patterns

- Generic secrets/tokens (20+ characters)
- AWS credentials (`AKIA...`, `aws_access_key_id`, etc.)
- OpenAI API keys (`sk-...`)
- Google API keys (`AIza...`)
- GitHub Personal Access Tokens (`ghp_...`)
- Bearer tokens
- Private keys (`-----BEGIN PRIVATE KEY-----`)
- Hardcoded values from `.env` files

## Requirements

- Python 3
- Git repository
- No external dependencies

## Example Output

```
🚨 SECURITY WARNING: Potential secrets detected in staged files!

  Pattern-based detections:
    ❌ src/config.js:12 - Found potential OpenAI API key

  Hardcoded .env values detected:
    ❌ src/client.ts:45 - Found hardcoded value from .env key 'DATABASE_URL'

⚠️  Please remove secrets before committing.
💡 Use environment variables at runtime instead of hardcoding values from .env.
💡 Consider using a secrets manager for sensitive credentials.
```

## License

MIT
