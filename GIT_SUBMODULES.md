# Git Submodules Guide - Wahb Monorepo

This document covers common commands for working with git submodules in the Wahb Project.

---

## Quick Command Summary

| Command | Task |
|---------|------|
| `git submodule update --init --recursive` | Initialize all submodules (first time setup) |
| `git submodule foreach 'git checkout main'` | Checkout main branch for all submodules |
| `git clone --recurse-submodules <url>` | Clone repo with all submodules in one command |
| `git submodule status` | View status of all submodules |
| `git submodule foreach 'echo $path'` | List all submodule paths |
| `git submodule update --remote --merge` | Update all submodules to latest and merge |
| `git submodule update --remote` | Update without merging (fetch only) |
| `git submodule foreach git pull origin main` | Pull latest changes for all submodules |
| `git submodule update --init --recursive` | Reset all submodules to committed state |
| `git submodule foreach 'git branch'` | Show current branch in all submodules |
| `git status` | See which submodules have changed |
| `git diff Aggregation-Service` | See changes inside a submodule |
| `git submodule foreach 'git log -1 --oneline'` | See last commit for all submodules |
| `git submodule foreach <command>` | Run any command in all submodules |
| `git submodule deinit -f <path>` | Remove a submodule |
| `git rm -f <path>` | Remove submodule reference from git |

---

## First-Time Setup

### Clone and Initialize All Submodules
When you first clone the Wahb monorepo, the submodule folders will be empty. Run:

```bash
git submodule update --init --recursive
```

**What it does:**
- `--init`: Initializes all submodules listed in `.gitmodules`
- `--recursive`: Also initializes any nested submodules
- Clones all 5 services into their respective folders

### Alternative: Clone in One Command
If you haven't cloned the repo yet:

```bash
git clone --recurse-submodules https://github.com/SalehAlobaylan/Wahb-Project.git
```

---

## Checking Submodule Status

### View Submodule Status
```bash
git submodule status
```

**Output indicators:**
- `1a53993... Aggregation-Service` (no prefix) → Submodule is initialized and at the correct commit
- `-1a53993... Aggregation-Service` (minus sign) → Submodule not initialized
- `+1a53993... Aggregation-Service` (plus sign) → Submodule has uncommitted changes
- `U1a53993... Aggregation-Service` (U) → Submodule has merge conflicts

### List All Submodules
```bash
git submodule foreach 'echo $path'
```

---

## Updating Submodules

### Update All Submodules to Latest
```bash
git submodule update --remote --merge
```

**What it does:**
- Fetches latest changes from each submodule's remote
- Merges them into the submodule
- Updates the main repo to point to new submodule commits

### Update Without Merging
```bash
git submodule update --remote
```

### Pull Latest Changes for All Submodules
```bash
git submodule foreach git pull origin main
```

**Note:** Replace `main` with `master` or the correct branch name for each service.

### Update All Submodules to Committed State
```bash
git submodule update --init --recursive
```

This resets all submodules to the commits specified in the main repo.

---

## Working Within Submodules

### Navigate to a Submodule
```bash
cd Aggregation-Service
```

### Make Changes in a Submodule
1. Navigate to the submodule directory
2. Make your changes
3. Commit within the submodule:
   ```bash
   git add .
   git commit -m "Your commit message"
   git push origin main
   ```
4. Go back to main repo and update the reference:
   ```bash
   cd ..
   git add Aggregation-Service
   git commit -m "Update Aggregation-Service submodule"
   ```

### Check Submodule Branch
```bash
git submodule foreach 'git branch'
```

---

## Common Scenarios

### Scenario 1: Someone Updated the Main Repo
You pull the main repo and see submodule folders are outdated:

```bash
git pull origin main
git submodule update --init --recursive
```

### Scenario 2: You Want to Work on a Specific Service
Navigate to the service, check out a branch, and work:

```bash
cd Content-Management-System
git checkout -b feature/new-feature
# Make changes, commit, push
```

### Scenario 3: Submodule is in Detached HEAD State
If you see `(HEAD detached at 1a53993)`:

```bash
cd Aggregation-Service
git checkout main
# OR create a new branch from the detached commit
git checkout -b my-feature-branch
```

### Scenario 4: Remove All Submodules
**Warning:** This deletes all submodule code and references.

```bash
# Remove each submodule
git submodule deinit -f Aggregation-Service
git rm -f Aggregation-Service
rm -rf .git/modules/Aggregation-Service

# Repeat for all submodules...
```

---

## Viewing Changes

### See Which Submodules Changed
```bash
git status
```

Output will show:
```
modified:   Aggregation-Service (new commits)
modified:   Wahb-Platform (new commits)
```

### See Changes Inside a Submodule
```bash
git diff Aggregation-Service
```

### See All Submodule Logs
```bash
git submodule foreach 'git log -1 --oneline'
```

---

## Troubleshooting

### Submodule Folder is Empty
```bash
git submodule update --init --recursive
```

### Submodule Shows Modified but No Changes
If `git status` shows a submodule as modified but you didn't change anything:
```bash
cd Aggregation-Service
git checkout main
cd ..
git add Aggregation-Service
```

### Submodule has Merge Conflicts
```bash
cd Content-Management-System
# Resolve conflicts
git add .
git commit
cd ..
```

### Fetch All Submodule Updates Without Merging
```bash
git submodule update --remote --no-merge
```

---

## Quick Reference Card

| Task | Command |
|------|---------|
| **First time setup** | `git submodule update --init --recursive` |
| **Update all submodules** | `git submodule update --remote --merge` |
| **Check status** | `git submodule status` |
| **Pull latest in all** | `git submodule foreach git pull origin main` |
| **Run command in all** | `git submodule foreach <command>` |
| **Reset to committed state** | `git submodule update --init --recursive` |
| **See submodule branches** | `git submodule foreach git branch` |
| **See submodule logs** | `git submodule foreach git log -1 --oneline` |

---

## Notes

- The main repo stores **commit references**, not the actual code
- Each submodule is a separate git repository
- When you commit in a submodule, you must also commit the submodule reference in the main repo
- Always push submodule changes before updating the main repo reference
- The `.gitmodules` file defines the submodule structure
- Submodule data is stored in `.git/modules/` directory

---

## Wahb Project Submodules

| Submodule | Path | Remote Repository |
|-----------|------|-------------------|
| Aggregation-Service | `./Aggregation-Service` | `https://github.com/SalehAlobaylan/Aggregation-Service` |
| CRM-Service | `./CRM-Service` | `https://github.com/SalehAlobaylan/CRM-Service` |
| Content-Management-System | `./Content-Management-System` | `https://github.com/SalehAlobaylan/Content-Management-System` |
| Platform-Console | `./Platform-Console` | `https://github.com/SalehAlobaylan/Platform-Console` |
| Wahb-Platform | `./Wahb-Platform` | `https://github.com/SalehAlobaylan/Wahb-Platform` |
