# Plugin Quick Start

## Plugin Structure

Your FHE Challenge Solver plugin is now ready! Here's the structure:

```
CC/
├── .claude-plugin/
│   ├── plugin.json          # Plugin manifest (required)
│   └── QUICKSTART.md        # This file
├── agents/
│   └── fhe-challenge-autopilot.md
├── skills/
│   ├── challenge-understanding/SKILL.md
│   ├── function-approximation/SKILL.md
│   ├── encrypted-computation/SKILL.md
│   ├── ml-pipeline/SKILL.md
│   ├── openfhe-mastery/SKILL.md
│   └── solution-engineering/SKILL.md
├── commands/
│   └── fhe-solve.md
├── CLAUDE.md                # Repository context for Claude
└── README.md                # Plugin documentation
```

## Testing Locally

Test your plugin without installing:

```bash
# From any directory, use --plugin-dir flag
claude --plugin-dir /home/jiaq/Research/Code/CC

# Your command will be available as:
/fhe-challenge-solver:fhe-solve <challenge_path>
```

## Publishing to GitHub

1. **Initialize git repository** (if not already done):
   ```bash
   cd /home/jiaq/Research/Code/CC
   git init
   git add .
   git commit -m "Initial commit: FHE Challenge Solver plugin"
   ```

2. **Create GitHub repository**:
   - Go to github.com and create a new repository
   - Name it `fhe-challenge-solver` or similar

3. **Push to GitHub**:
   ```bash
   git remote add origin https://github.com/yourusername/fhe-challenge-solver.git
   git branch -M main
   git push -u origin main
   ```

4. **Add topics** on GitHub (optional):
   - claude-code-plugin
   - fhe
   - homomorphic-encryption
   - openfhe

## Installing from GitHub

Once published, anyone can install:

```bash
# Add your marketplace
/plugin marketplace add yourusername/fhe-challenge-solver

# Install the plugin
/plugin install fhe-challenge-solver
```

## Command Namespacing

Your slash command is automatically namespaced:
- File: `commands/fhe-solve.md`
- Plugin name: `fhe-challenge-solver`
- Available as: `/fhe-challenge-solver:fhe-solve`

If someone has the plugin installed globally, they can use it in any project directory.

## Updating the Plugin

When you make changes:

```bash
git add .
git commit -m "Update: description of changes"
git push

# Update version in .claude-plugin/plugin.json
# Users can update with: /plugin update fhe-challenge-solver
```

## Version Guidelines

Follow semantic versioning in `plugin.json`:
- **1.0.0** → **1.0.1**: Bug fixes, minor improvements
- **1.0.0** → **1.1.0**: New features, new skills
- **1.0.0** → **2.0.0**: Breaking changes to command interface

## What Users Get

When someone installs your plugin, they get:
- ✅ The `/fhe-challenge-solver:fhe-solve` command
- ✅ The `fhe-challenge-autopilot` agent
- ✅ All 6 specialized skills
- ✅ Ability to use your agents and skills in their own workflows

## Best Practices

1. **Keep CLAUDE.md updated** - This helps Claude understand your plugin
2. **Document in README.md** - Clear instructions for users
3. **Test locally first** - Use `--plugin-dir` before publishing
4. **Version carefully** - Update version in plugin.json with each release
5. **Add examples** - Include sample challenges or usage patterns

## Support

For questions about Claude Code plugins:
- Docs: https://code.claude.com/docs/en/plugins
- GitHub: https://github.com/anthropics/claude-code
