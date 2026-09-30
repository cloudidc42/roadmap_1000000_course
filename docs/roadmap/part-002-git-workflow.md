# Part 002: Git & Version Control สำหรับ Team
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Beginner
> **Steps:** 11-20
> **เวลาโดยประมาณ:** 6 ชั่วโมง
> **Prerequisites:** Part 001 (Linux Basics)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. GitFlow workflow และการทำงานร่วมกันใน team
2. Trunk-based development สำหรับ CI/CD ที่ rapid
3. Branch naming conventions ที่เป็นมาตรฐาน
4. Git hooks: pre-commit, pre-push, commit-msg
5. Conventional Commits specification พร้อมตัวอย่าง
6. Semantic Versioning (MAJOR.MINOR.PATCH)
7. Git tags สำหรับ release management
8. .gitignore สำหรับ Next.js + Node.js project
9. .gitconfig พร้อม useful aliases
10. Git conflict resolution workflow และ code review checklist

---

## 📖 ทฤษฎีและแนวคิด

### GitFlow Workflow Diagram

```
main ──────────────────●─────────────────────────●──────────
                      /│                          │\
                     / │                          │ \
develop ────●───────●  │                     ●───●  ●────────
           /         \ │                    / \
          /           \│                   /   \
feature/ ●─────────────●   hotfix/ ●──────●     ●
feature/sos-alert       \          \
                         release/   ●──────●
                         1.0.0

Branch ประเภทต่างๆ:
┌─────────────────────────────────────────────────────────┐
│  main       → production code เท่านั้น (protected)      │
│  develop    → integration branch สำหรับ features        │
│  feature/*  → feature development branches              │
│  release/*  → release preparation branches              │
│  hotfix/*   → urgent production fixes                   │
└─────────────────────────────────────────────────────────┘
```

### Trunk-Based Development Diagram

```
main/trunk ──────●──●──●──●──●──●──●──●──────────────────
                 │     │     │     │
                feat  feat  fix  feat
                (short-lived branches < 24h)

CI/CD Pipeline:
push → lint → test → build → deploy to staging → deploy to prod
  ↑___________________________________________|
              feedback loop (< 10 minutes)
```

### Conventional Commits Format

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]

Types:
  feat:     New feature
  fix:      Bug fix
  docs:     Documentation only
  style:    Formatting (no logic change)
  refactor: Code restructuring (no feature/fix)
  perf:     Performance improvement
  test:     Adding/fixing tests
  chore:    Build process, dependencies, tooling
  ci:       CI/CD changes
  revert:   Reverting a previous commit
  build:    Changes to build system

ตัวอย่าง:
  feat(auth): add JWT token refresh mechanism
  fix(sos): correct geolocation accuracy calculation
  docs(api): update REST API documentation for v2
  perf(feed): add Redis cache for social feed query
  chore(deps): upgrade Next.js to 15.1.0
  feat!: BREAKING CHANGE — redesign auth API
```

---

## ⚙️ Environment Setup

```bash
# ตรวจสอบ Git version
git --version
# Expected: git version 2.43.x

# ติดตั้ง Git ล่าสุด
sudo add-apt-repository ppa:git-core/ppa -y
sudo apt update
sudo apt install -y git

# ติดตั้ง tools เพิ่มเติม
sudo apt install -y gh        # GitHub CLI
npm install -g commitizen cz-conventional-changelog @commitlint/cli @commitlint/config-conventional husky lint-staged

# ตรวจสอบ
git --version         # 2.43.x
gh --version          # gh version 2.x.x
commitizen --version
```

---

## 🛠️ Step-by-Step Implementation

### Step 11: Global Git Configuration

```bash
# ─── ตั้งค่า Git Identity ──────────────────────────────────
git config --global user.name "Your Name"
git config --global user.email "you@chuaikan.com"

# ─── Default Branch ────────────────────────────────────────
git config --global init.defaultBranch main

# ─── Editor ────────────────────────────────────────────────
git config --global core.editor "vim"
# หรือ VS Code:
# git config --global core.editor "code --wait"

# ─── Diff Tool ─────────────────────────────────────────────
git config --global diff.tool vimdiff
git config --global merge.tool vimdiff

# ─── Color Output ──────────────────────────────────────────
git config --global color.ui auto
git config --global color.branch auto
git config --global color.diff auto
git config --global color.status auto
```

### .gitconfig พร้อม Useful Aliases

```bash
cat > ~/.gitconfig << 'EOF'
[user]
    name = Your Name
    email = you@chuaikan.com

[core]
    editor = vim
    autocrlf = input
    whitespace = fix,-indent-with-non-tab,trailing-space,cr-at-eol
    excludesfile = ~/.gitignore_global

[init]
    defaultBranch = main

[color]
    ui = auto

[pull]
    rebase = false

[push]
    default = current
    followTags = true

[fetch]
    prune = true

[diff]
    tool = vimdiff
    algorithm = histogram

[merge]
    tool = vimdiff
    conflictstyle = diff3

[rebase]
    autosquash = true

[alias]
    # Status & Info
    st = status
    s  = status --short --branch
    lg = log --oneline --graph --decorate --all
    ll = log --pretty=format:"%C(yellow)%h%Cred%d %Creset%s%Cblue [%cn]" --decorate --numstat
    last = log -1 HEAD --stat

    # Branching
    br = branch
    bra = branch -a
    co = checkout
    cob = checkout -b
    sw = switch
    swc = switch -c

    # Committing
    c = commit
    ca = commit --amend
    can = commit --amend --no-edit

    # Staging
    a = add
    aa = add --all
    ap = add --patch       # interactive staging

    # Diff
    d = diff
    ds = diff --staged
    dc = diff --cached

    # Push/Pull
    p = push
    pf = push --force-with-lease    # safe force push
    pl = pull
    up = pull --rebase --prune

    # Stash
    ss = stash
    sl = stash list
    sp = stash pop
    sd = stash drop

    # Cleanup
    cleanup = "!git branch --merged | grep -v '\\*\\|main\\|develop' | xargs -n 1 git branch -d"
    gone = "!git fetch -p && git branch -vv | awk '/: gone]/{print $1}' | xargs git branch -D"

    # Useful
    aliases = config --get-regexp alias
    undo = reset HEAD~1 --soft     # undo last commit (keep changes)
    unstage = reset HEAD --
    discard = checkout --
    tags = tag -l --sort=-version:refname
    contributors = shortlog --summary --numbered --no-merges

    # Find
    find = log --all --grep

    # GitHub CLI shortcuts (ต้องติดตั้ง gh)
    pr = "!gh pr create"
    prl = "!gh pr list"
    prv = "!gh pr view"

[credential]
    helper = cache --timeout=3600

[rerere]
    enabled = true

[help]
    autocorrect = 20
EOF

# ทดสอบ aliases
git lg        # ดู log แบบ graph
git s         # ดู status แบบสั้น
git bra       # ดู branches ทั้งหมด
```

### Step 12: GitFlow Setup และ Branch Naming

```bash
# ─── Initialize GitFlow ────────────────────────────────────
# ติดตั้ง git-flow
sudo apt install -y git-flow

# Initialize project ด้วย GitFlow
cd /opt/chuaikan/app
git init
git flow init -d     # -d = use defaults

# ─── Branch Naming Conventions ────────────────────────────
# feature/: New features
git flow feature start sos-alert-system
# Creates: feature/sos-alert-system from develop

git flow feature start user-profile-v2
# Creates: feature/user-profile-v2

# fix/: Bug fixes (non-critical)
git checkout develop
git checkout -b fix/login-validation-error
git checkout -b fix/image-upload-crash

# hotfix/: Critical production fixes
git flow hotfix start fix-payment-gateway
# Creates: hotfix/fix-payment-gateway from main

# release/: Release preparation
git flow release start 1.2.0
# Creates: release/1.2.0 from develop

# chore/: Maintenance tasks
git checkout develop
git checkout -b chore/upgrade-dependencies
git checkout -b chore/cleanup-unused-imports

# ─── Branch Protection Rules (GitHub) ─────────────────────
# ตั้งค่าผ่าน GitHub UI หรือ CLI:
gh api repos/chuaikan/chuaikan-app/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["ci/test","ci/lint"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null
```

### Step 13: .gitignore สำหรับ Next.js + Node.js

```bash
cat > /opt/chuaikan/app/.gitignore << 'EOF'
# ─── Dependencies ─────────────────────────────────────────
node_modules/
.pnp
.pnp.js
.yarn/install-state.gz

# ─── Next.js Build ────────────────────────────────────────
.next/
out/
build/
dist/

# ─── Environment Variables (NEVER commit these!) ──────────
.env
.env.local
.env.development.local
.env.test.local
.env.production.local
.env.production
.env.staging

# ─── Debug Logs ───────────────────────────────────────────
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.pnpm-debug.log*
lerna-debug.log*

# ─── Cache ────────────────────────────────────────────────
.eslintcache
.stylelintcache
.turbo

# ─── OS Files ─────────────────────────────────────────────
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db
desktop.ini
*~

# ─── Editor Files ─────────────────────────────────────────
.vscode/
!.vscode/extensions.json
!.vscode/settings.json
.idea/
*.swp
*.swo
*.suo
*.ntvs*
*.njsproj
*.sln
*.sw?
.project
.classpath

# ─── Testing ──────────────────────────────────────────────
coverage/
.nyc_output
*.lcov
/test-results/
/playwright-report/
/playwright/.cache/

# ─── Storybook ────────────────────────────────────────────
storybook-static/

# ─── Uploads (use cloud storage in production) ────────────
public/uploads/
uploads/

# ─── Certificates ─────────────────────────────────────────
*.pem
*.p12
*.key
*.crt
*.cert

# ─── Docker ───────────────────────────────────────────────
.dockerignore

# ─── Misc ─────────────────────────────────────────────────
*.tsbuildinfo
next-env.d.ts
.vercel
.netlify
EOF
```

```bash
# Global .gitignore (สำหรับ personal files)
cat > ~/.gitignore_global << 'EOF'
# macOS
.DS_Store
.AppleDouble
.LSOverride

# Linux
*~
.directory

# VS Code
.vscode/settings.json

# JetBrains
.idea/

# Logs
*.log

# Keys & Secrets
*.pem
*.key
id_rsa
id_ed25519
EOF

git config --global core.excludesfile ~/.gitignore_global
```

### Step 14: Git Hooks

```bash
# ─── ติดตั้ง Husky (modern git hooks) ────────────────────
cd /opt/chuaikan/app
npm install --save-dev husky lint-staged

# Initialize husky
npx husky init
# สร้างไฟล์ .husky/pre-commit

# ─── commitlint setup ─────────────────────────────────────
npm install --save-dev @commitlint/cli @commitlint/config-conventional

cat > commitlint.config.js << 'EOF'
/** @type {import('@commitlint/types').UserConfig} */
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'perf', 'test', 'chore', 'ci', 'revert', 'build'
    ]],
    'scope-case': [2, 'always', 'lower-case'],
    'subject-max-length': [2, 'always', 100],
    'body-max-line-length': [2, 'always', 150],
  },
};
EOF

# Hook: commit-msg (validate commit message format)
cat > .husky/commit-msg << 'EOF'
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

npx --no -- commitlint --edit "$1"
EOF
chmod +x .husky/commit-msg
```

```bash
# ─── Hook: pre-commit (lint + format check) ───────────────
cat > .husky/pre-commit << 'HOOK'
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "🔍 Running pre-commit checks..."

# Run lint-staged (lint only changed files)
npx lint-staged

# Check for secrets/sensitive data
echo "🔐 Checking for secrets..."
git diff --cached --name-only | while read -r file; do
    if [ -f "$file" ]; then
        if grep -qE "(password|secret|api_key|private_key|token)\s*=\s*['\"][^'\"]{8,}" "$file" 2>/dev/null; then
            echo "❌ Possible secret detected in: $file"
            exit 1
        fi
    fi
done

echo "✅ Pre-commit checks passed!"
HOOK
chmod +x .husky/pre-commit
```

```bash
# ─── Hook: pre-push (run tests) ────────────────────────────
cat > .husky/pre-push << 'HOOK'
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "🧪 Running tests before push..."

# Get current branch
BRANCH=$(git rev-parse --abbrev-ref HEAD)

# Skip tests for WIP branches
if echo "$BRANCH" | grep -qE "^wip/"; then
    echo "⚠️  WIP branch detected, skipping tests"
    exit 0
fi

# Run tests
npm run test:ci
EXIT_CODE=$?

if [ $EXIT_CODE -ne 0 ]; then
    echo "❌ Tests failed! Fix tests before pushing."
    echo "   To skip: git push --no-verify (ไม่แนะนำ)"
    exit 1
fi

echo "✅ All tests passed!"
HOOK
chmod +x .husky/pre-push
```

```json
// package.json — เพิ่ม lint-staged config
{
  "scripts": {
    "lint": "next lint",
    "lint:fix": "next lint --fix",
    "test": "jest",
    "test:ci": "jest --ci --coverage",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  },
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,css,md,yml,yaml}": [
      "prettier --write"
    ],
    "*.sql": [
      "sql-formatter --fix"
    ]
  }
}
```

### Step 15: Semantic Versioning และ Git Tags

```bash
# ─── Semantic Versioning ──────────────────────────────────
# MAJOR.MINOR.PATCH
# MAJOR: breaking changes (incompatible API changes)
# MINOR: new features (backward compatible)
# PATCH: bug fixes (backward compatible)
#
# ตัวอย่าง:
# 1.0.0 → Initial release
# 1.0.1 → Bug fix: sos alert location accuracy
# 1.1.0 → New feature: real-time chat
# 2.0.0 → Breaking: new API structure

# Pre-release versions:
# 1.0.0-alpha.1   → early testing
# 1.0.0-beta.2    → feature complete, testing
# 1.0.0-rc.1      → release candidate

# ─── Git Tags ─────────────────────────────────────────────
# สร้าง annotated tag (recommended สำหรับ releases)
git tag -a v1.0.0 -m "Release v1.0.0

Features:
- User registration and authentication
- SOS alert system with geolocation
- Social feed with real-time updates
- Image upload with optimization

Bug Fixes:
- Fixed login redirect loop
- Corrected SOS radius calculation"

# ดู tags
git tag -l
git tag -l "v1.*"                    # filter pattern
git show v1.0.0                      # ดูรายละเอียด tag

# Push tags ไป remote
git push origin v1.0.0               # push specific tag
git push origin --tags               # push all tags

# ─── Release Script ────────────────────────────────────────
cat > /opt/chuaikan/scripts/release.sh << 'SCRIPT'
#!/bin/bash
# release.sh — สร้าง release tag อัตโนมัติ
set -euo pipefail

VERSION=$1
if [ -z "$VERSION" ]; then
    echo "Usage: $0 <version>"
    echo "Example: $0 1.2.0"
    exit 1
fi

# ตรวจสอบ format
if ! echo "$VERSION" | grep -qE "^[0-9]+\.[0-9]+\.[0-9]+(-[a-z]+\.[0-9]+)?$"; then
    echo "Invalid version format. Use MAJOR.MINOR.PATCH"
    exit 1
fi

TAG="v${VERSION}"

# ตรวจสอบว่า tag ยังไม่มี
if git rev-parse "$TAG" >/dev/null 2>&1; then
    echo "Tag $TAG already exists!"
    exit 1
fi

echo "Creating release $TAG..."

# อัพเดต version ใน package.json
npm version "$VERSION" --no-git-tag-version

# Commit version bump
git add package.json package-lock.json
git commit -m "chore(release): bump version to $VERSION"

# สร้าง tag
git tag -a "$TAG" -m "Release $TAG

See CHANGELOG.md for details"

echo "✅ Created tag $TAG"
echo "To push: git push && git push --tags"
SCRIPT
chmod +x /opt/chuaikan/scripts/release.sh
```

### Step 16: GitFlow Complete Workflow

```bash
# ─── Feature Development Workflow ─────────────────────────
# 1. เริ่ม feature ใหม่
git flow feature start sos-realtime

# 2. เขียน code...
echo "// SOS real-time component" > src/components/SosAlert.tsx

# 3. Commit ด้วย Conventional Commits
git add src/components/SosAlert.tsx
git commit -m "feat(sos): add real-time SOS alert component

- Implement WebSocket connection for SOS alerts
- Add geolocation support with accuracy fallback
- Display alert radius on interactive map

Closes #42"

# 4. Push feature branch
git flow feature publish sos-realtime

# 5. สร้าง Pull Request
gh pr create \
  --title "feat(sos): add real-time SOS alert component" \
  --body "## Summary
- Real-time SOS alert via WebSocket
- Geolocation with accuracy fallback
- Interactive map with alert radius

## Test Plan
- [ ] Unit tests for SOS service
- [ ] E2E test: send and receive SOS alert
- [ ] Test geolocation on mobile

Closes #42" \
  --base develop \
  --reviewer "@teammate1,@teammate2"

# 6. หลัง PR merged:
git flow feature finish sos-realtime

# ─── Release Workflow ──────────────────────────────────────
# 1. เริ่ม release
git flow release start 1.2.0

# 2. อัพเดต version และ CHANGELOG
npm version 1.2.0 --no-git-tag-version
# แก้ไข CHANGELOG.md

git commit -am "chore(release): prepare v1.2.0"

# 3. Finish release (merge ไป main และ develop)
git flow release finish 1.2.0
# ป้อน tag message: "Release v1.2.0"

# 4. Push ทุก branches และ tags
git push origin main develop
git push origin --tags

# ─── Hotfix Workflow ──────────────────────────────────────
# 1. เริ่ม hotfix (from main)
git flow hotfix start fix-sos-null-pointer

# 2. แก้ bug
# ...edit files...
git commit -am "fix(sos): handle null user location gracefully

Prevent NullPointerException when user denies location permission.
Returns default location (Bangkok) with accuracy warning.

Fixes #99"

# 3. Finish hotfix (merge ไป main และ develop)
git flow hotfix finish fix-sos-null-pointer
git push origin main develop --tags
```

### Step 17: Git Conflict Resolution Workflow

```bash
# ─── เกิด Conflict เมื่อ merge ────────────────────────────
git merge feature/user-profile
# Auto-merging src/api/users.ts
# CONFLICT (content): Merge conflict in src/api/users.ts

# ดู conflicted files
git status
# Output:
# both modified: src/api/users.ts

# ─── ดู conflict markers ───────────────────────────────────
cat src/api/users.ts
# <<<<<<< HEAD (current branch)
# export const getUser = async (id: string) => {
#   return db.users.findUnique({ where: { id } });
# =======
# export const getUser = async (id: string) => {
#   const user = await prisma.user.findUnique({ where: { id } });
#   return user;
# >>>>>>> feature/user-profile

# ─── วิธีแก้ Conflict ─────────────────────────────────────
# Option 1: Manual edit ไฟล์ที่ conflict
vim src/api/users.ts
# ลบ conflict markers และเลือก version ที่ถูกต้อง
# หรือ merge ทั้งสอง version เข้าด้วยกัน

# Option 2: ใช้ vimdiff
git mergetool src/api/users.ts

# Option 3: ใช้ VS Code
code src/api/users.ts    # VS Code มี GUI สำหรับ resolve conflicts

# หลัง resolve แล้ว:
git add src/api/users.ts
git commit -m "merge: resolve conflict in users.ts API

Kept database abstraction from feature/user-profile
while maintaining backward compatibility from main"

# ─── Rebase แทน Merge (cleaner history) ───────────────────
git checkout feature/sos-update
git rebase develop

# ถ้า conflict:
# แก้ conflict ใน editor
git add .
git rebase --continue
# (ทำซ้ำจนกว่าจะ finish)

# ถ้าต้องการยกเลิก rebase:
git rebase --abort
```

### Step 18: Code Review Checklist

```markdown
# Code Review Checklist สำหรับ chuaikan.com

## 📋 Pre-Review (Author)
- [ ] Self-review ก่อน request review
- [ ] เพิ่ม description และ test plan ใน PR
- [ ] Tests ผ่านทั้งหมด (CI green)
- [ ] Code ไม่มี debug statements (console.log ที่ไม่ต้องการ)
- [ ] ไม่มี hardcoded secrets หรือ API keys
- [ ] PR size ไม่ใหญ่เกิน 400 lines ที่เปลี่ยนแปลง

## 🔍 Code Quality
- [ ] Code อ่านเข้าใจง่าย
- [ ] Variable/function names สื่อความหมาย
- [ ] ไม่มี code ซ้ำซ้อน (DRY principle)
- [ ] Functions มีขนาดเล็กและทำหน้าที่เดียว (SRP)
- [ ] ไม่มี TODO/FIXME ที่ไม่มี issue ผูก

## 🔒 Security
- [ ] Input validation ครบถ้วน
- [ ] SQL injection prevention (ใช้ parameterized queries)
- [ ] XSS prevention (sanitize user input)
- [ ] Authorization check ถูกต้อง (ไม่ใช่แค่ authentication)
- [ ] ไม่ expose sensitive data ใน response

## ⚡ Performance
- [ ] ไม่มี N+1 query problem
- [ ] Indexes ที่จำเป็นมีอยู่แล้ว
- [ ] Large responses มี pagination
- [ ] Images มี optimization

## 🧪 Testing
- [ ] Unit tests สำหรับ business logic
- [ ] Happy path และ edge cases มี test
- [ ] Error handling มี test
- [ ] Test coverage > 80%
```

```bash
# ─── GitHub CLI สำหรับ Code Review ────────────────────────
# ดู PRs ที่รอ review
gh pr list --review-requested @me

# เปิด PR เพื่อ review
gh pr view 42 --web

# Review ผ่าน CLI
gh pr review 42 --comment "LGTM! Minor suggestion: ..."
gh pr review 42 --approve
gh pr review 42 --request-changes --body "Please fix the N+1 query in line 45"

# Merge PR
gh pr merge 42 --squash --delete-branch
```

### Step 19: CHANGELOG.md Template

```bash
cat > /opt/chuaikan/app/CHANGELOG.md << 'EOF'
# Changelog

All notable changes to chuaikan.com will be documented in this file.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/)
and [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.2.0] - 2024-09-30
### Added
- Real-time SOS alert system with WebSocket
- User location sharing in SOS alerts
- Push notifications for nearby SOS alerts

### Changed
- Improved social feed algorithm performance
- Updated image upload to support WebP format

### Fixed
- Fixed login redirect loop (#87)
- Corrected SOS alert radius calculation (#91)

### Security
- Updated bcrypt to latest version
- Added rate limiting to auth endpoints

## [1.1.0] - 2024-08-15
### Added
- User profile customization
- Follow/unfollow system
- Image upload with automatic resizing

## [1.0.0] - 2024-07-01
### Added
- Initial release
- User registration and authentication
- Basic social feed
- SOS alert system (basic)

[Unreleased]: https://github.com/chuaikan/chuaikan-app/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/chuaikan/chuaikan-app/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/chuaikan/chuaikan-app/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/chuaikan/chuaikan-app/releases/tag/v1.0.0
EOF
```

### Step 20: GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '22'

jobs:
  lint:
    name: Lint & Format Check
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Check Prettier formatting
        run: npm run format:check

      - name: Validate commit messages
        uses: wagoid/commitlint-github-action@v5

  test:
    name: Test
    runs-on: ubuntu-24.04
    needs: lint

    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: chuaikan_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run database migrations
        run: npm run db:migrate
        env:
          DATABASE_URL: postgresql://test_user:test_password@localhost:5432/chuaikan_test

      - name: Run tests
        run: npm run test:ci
        env:
          DATABASE_URL: postgresql://test_user:test_password@localhost:5432/chuaikan_test
          REDIS_URL: redis://localhost:6379

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  build:
    name: Build
    runs-on: ubuntu-24.04
    needs: test
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build Next.js
        run: npm run build
        env:
          NODE_ENV: production

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: nextjs-build
          path: .next/
```

---

## 🔧 Configuration Files

### .editorconfig

```ini
# .editorconfig — ตั้งค่า editor ให้ consistent ทั้ง team
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 2

[*.md]
trim_trailing_whitespace = false

[Makefile]
indent_style = tab

[*.{py,pyi}]
indent_size = 4
```

---

## 🧪 Testing

```bash
# ─── ทดสอบ Git Hooks ──────────────────────────────────────
# ทดสอบ commit-msg hook (ควร fail)
git commit --allow-empty -m "bad commit message"
# Expected: ✖  subject may not be empty [subject-empty]

# ทดสอบ commit ที่ถูกต้อง
git commit --allow-empty -m "feat(sos): test conventional commit"
# Expected: ✔  Commit message is valid!

# ─── ทดสอบ pre-commit hook ────────────────────────────────
# สร้างไฟล์ที่มี lint error
echo "var x=1" > test.js
git add test.js
git commit -m "test: lint check"
# Expected: ESLint error → commit blocked

# ─── ทดสอบ GitFlow ────────────────────────────────────────
git flow feature start test-feature
git flow feature finish test-feature
git log --oneline -5
# Expected: merge commit แสดงใน develop branch

# ─── ทดสอบ Tag ────────────────────────────────────────────
git tag -a v0.0.1-test -m "Test tag"
git show v0.0.1-test
git tag -d v0.0.1-test   # ลบ test tag
```

---

## ❌ Common Errors & Solutions

### Error 1: Husky hooks ไม่ทำงาน

```
❌ Error: hint: The '.husky/pre-commit' hook was ignored because
it's not set as executable.

✅ Solution:
chmod +x .husky/pre-commit
chmod +x .husky/commit-msg
chmod +x .husky/pre-push

# หรือ initialize ใหม่
npx husky install
```

### Error 2: Commit message ไม่ผ่าน

```
❌ Error: ✖  type must be one of [feat, fix, docs, ...]

✅ Solution:
# ใช้ commitizen สำหรับ interactive commit
npm install -g commitizen
git cz
# จะมี wizard ให้เลือก type, scope, description

# หรือแก้ message ที่ commit ล่าสุด
git commit --amend
```

### Error 3: Merge conflict ซับซ้อน

```
❌ Error: CONFLICT (content): Merge conflict in package-lock.json

✅ Solution:
# package-lock.json conflicts → regenerate
git checkout --theirs package-lock.json
npm install
git add package-lock.json
git commit -m "fix: regenerate package-lock.json after merge"

# หรือใช้ rerere (reuse recorded resolution)
git config --global rerere.enabled true
# Git จะจำวิธีแก้ conflict และ apply อัตโนมัติครั้งต่อไป
```

### Error 4: Push ถูก reject เพราะ force-push

```
❌ Error: ! [rejected] main -> main (non-fast-forward)

✅ Solution:
# ไม่ควร force push ไป main/develop!
# สำหรับ feature branch ของตัวเอง:
git push --force-with-lease origin feature/my-feature
# --force-with-lease ปลอดภัยกว่า --force
# จะ fail ถ้ามีคนอื่น push ไปก่อน

# ถ้าต้อง sync กับ remote:
git pull --rebase origin main
git push origin main
```

---

## 📊 Performance Benchmark

| ตัวชี้วัด | Without Workflow | With GitFlow + Hooks |
|---------|-----------------|---------------------|
| Merge conflicts/sprint | 8-10 | 2-3 |
| Bad commits to main | 15% | < 1% |
| CI failure rate | 35% | < 5% |
| Code review time | 2+ hours | 30-60 min |
| Release time | Manual 2h | Automated 15min |

---

## ✅ Checklist

- [ ] ตั้งค่า git config (name, email, aliases)
- [ ] สร้าง .gitignore สำหรับ Next.js + Node.js
- [ ] ติดตั้ง husky สำหรับ git hooks
- [ ] ตั้งค่า commitlint (conventional commits)
- [ ] ตั้งค่า lint-staged
- [ ] สร้าง pre-commit hook (lint + format)
- [ ] สร้าง pre-push hook (tests)
- [ ] สร้าง commit-msg hook (commitlint)
- [ ] Initialize GitFlow
- [ ] ตั้ง branch naming conventions
- [ ] ตั้งค่า branch protection บน GitHub
- [ ] สร้าง CHANGELOG.md
- [ ] ทดสอบ release workflow ด้วย tag
- [ ] ตั้งค่า GitHub Actions CI/CD
- [ ] ทดสอบ conflict resolution workflow

---

## 🔗 References

- [Conventional Commits](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)
- [GitFlow Workflow](https://nvie.com/posts/a-successful-git-branching-model/)
- [Trunk Based Development](https://trunkbaseddevelopment.com/)
- [Keep a Changelog](https://keepachangelog.com/)
- [Husky Documentation](https://typicode.github.io/husky/)
- [commitlint](https://commitlint.js.org/)
- [GitHub Actions](https://docs.github.com/en/actions)

---
*Part 002 | Road to 1,000,000 Users/Day | chuaikan.com*
