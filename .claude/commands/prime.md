#!/bin/bash
# Prime: Build comprehensive understanding of a codebase
# Usage: prime <folder_path>

set -e

if [ -z "$1" ]; then
    echo "Usage: prime <folder_path>"
    exit 1
fi

TARGET="$1"

if [ ! -d "$TARGET" ]; then
    echo "Error: '$TARGET' is not a valid directory"
    exit 1
fi

echo "=== PRIMING: $TARGET ==="
echo ""

cd "$TARGET"

# Check if it's a git repo
if [ -d ".git" ]; then
    echo "### Recent Activity"
    git log -10 --oneline 2>/dev/null || echo "(no git history)"
    echo ""

    echo "### Current Branch & Status"
    git branch --show-current 2>/dev/null || echo "(no branches)"
    git status --short 2>/dev/null | head -20 || echo "(clean)"
    echo ""

    echo "### Tracked Files"
    git ls-files 2>/dev/null | head -50 || echo "(no files)"
    echo ""
else
    echo "(Not a git repository - skipping git commands)"
    echo ""
fi

echo "### Directory Structure"
tree -L 3 -I 'node_modules|__pycache__|.git|dist|build|.venv|venv|vendor' 2>/dev/null || find . -maxdepth 3 -type f 2>/dev/null | head -50
echo ""

# Look for documentation files
echo "### Documentation Found"
for doc in PRD.md PLAN.md CLAUDE.md README.md README ARCHITECTURE.md CHANGELOG.md; do
    if [ -f "$doc" ]; then
        echo "- $doc"
    fi
done
echo ""

# Show file type distribution
echo "### File Types"
find . -type f -name "*.py" 2>/dev/null | wc -l | xargs -I{} echo "- Python: {} files"
find . -type f -name "*.js" 2>/dev/null | wc -l | xargs -I{} echo "- JavaScript: {} files"
find . -type f -name "*.ts" 2>/dev/null | wc -l | xargs -I{} echo "- TypeScript: {} files"
find . -type f -name "*.json" 2>/dev/null | wc -l | xargs -I{} echo "- JSON: {} files"
find . -type f -name "*.md" 2>/dev/null | wc -l | xargs -I{} echo "- Markdown: {} files"
echo ""

# Look for entry points and config files
echo "### Entry Points & Config"
for f in main.py index.ts app.py main.go main.rs main.js package.json pyproject.toml tsconfig.json Makefile Dockerfile; do
    if [ -f "$f" ]; then
        echo "- $f"
    fi
done
echo ""

echo "=== PRIMING COMPLETE ==="
