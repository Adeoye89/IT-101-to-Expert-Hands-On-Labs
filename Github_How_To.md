# GitHub 101 — Beginner to Advanced Hands-On Guide

> Learn by doing: create, add, edit, delete, fork, clone, branch, commit, push, pull, review, and recover.

## Contents
1. [Git vs GitHub](#1-git-vs-github)
2. [Create your account and repository](#2-create-your-account-and-repository)
3. [Add and edit files on the website](#3-add-and-edit-files-on-the-website)
4. [Delete and restore files](#4-delete-and-restore-files)
5. [Fork a repository](#5-fork-a-repository)
6. [Clone a repository](#6-clone-a-repository)
7. [Install Git and configure identity](#7-install-git-and-configure-identity)
8. [Git command workflow](#8-git-command-workflow)
9. [Branches and pull requests](#9-branches-and-pull-requests)
10. [Issues and collaboration](#10-issues-and-collaboration)
11. [Undo mistakes safely](#11-undo-mistakes-safely)
12. [Delete a repository safely](#12-delete-a-repository-safely)
13. [Security and troubleshooting](#13-security-and-troubleshooting)
14. [Practice project and checklist](#14-practice-project-and-checklist)

## 1. Git vs GitHub

**Git** is a version-control program that records changes to files. **GitHub** is a hosting and collaboration service for Git repositories. A **repository (repo)** is a project folder with version history. A **commit** is a saved checkpoint. A **branch** is a separate line of work. A **pull request (PR)** asks to merge changes from one branch into another. A **fork** is a repository copy under your own GitHub account.

Example: Your repository `IT-101-to-Expert-Hands-On-Labs` might contain:

```text
IT-101-to-Expert-Hands-On-Labs/
├── README.md
├── networking/
│   └── README.md
├── virtual-machines/
│   └── README.md
└── disk-partitioning/
    └── README.md
```

**Remember:** GitHub is not a substitute for a private backup, and public repositories are visible to everyone.

## 2. Create your account and repository

**Goal:** Create a place to store your IT training guides.

1. Visit https://github.com and choose **Sign up**. Verify your email and enable two-factor authentication.
2. Sign in, click the **+** menu near the top-right, and select **New repository**. You can also visit https://github.com/new.
3. Enter `IT-101-to-Expert-Hands-On-Labs` as the repository name.
4. Choose **Private** while practicing, or **Public** if you intend to share the lessons.
5. Select **Add a README file**. Optionally choose a license if you intend to grant reuse rights.
6. Click **Create repository**.
7. Verify that the repository home page displays `README.md`.

**Why:** The README is automatically shown on the repository landing page and can explain your project.

## 3. Add and edit files on the website

### Add a new file
1. Open your repository and navigate to the desired folder.
2. Select **Add file → Create new file**.
3. Enter `networking/README.md` in the filename box. The slash creates a folder path.
4. Type:

```markdown
# Networking 101
This folder contains networking exercises.
```

5. Select **Commit changes**; enter `Add networking lesson`.
6. Commit directly to the default branch for this simple exercise (or create a new branch if your repository requires review).
7. Confirm the file appears under `networking`.

### Upload a file from your computer
1. Open the target repository/folder.
2. Select **Add file → Upload files**.
3. Drag a file into the page or choose it with the file picker.
4. Enter a descriptive commit message, such as `Upload VM lab README`.
5. Select **Commit changes** or **Propose changes** as offered by GitHub.
6. Refresh the repository view and open the file to verify its contents.

### Edit an existing file
1. Open `README.md`.
2. Click the **pencil / Edit this file** control (you may need the `...` menu).
3. Change a heading or add a sentence.
4. Preview your changes if the preview is available.
5. Commit with `Clarify project introduction`.
6. Open **History** to see the new commit.

**Note:** GitHub's layout and button names can change slightly over time.

## 4. Delete and restore files

### Delete a file through the website
1. Open a disposable practice file, not your main README.
2. Open the file's menu and choose **Delete file** (trash icon or `...` menu).
3. Review the deletion and commit with `Remove practice file`.
4. Confirm it is no longer in the current file listing.

**What happened?** The file was removed from the latest version, but its content remains in earlier commits unless history is rewritten.

### Restore a deleted file
1. Open the repository **Commits** or file **History** view.
2. Find the last commit that contained the file.
3. Open the file at that earlier commit and copy its contents.
4. Recreate it with **Add file → Create new file** and commit.

For more complex recovery, use `git revert` (explained below). Never assume deleting a secret removes it from Git history; rotate exposed credentials immediately.

## 5. Fork a repository

**Goal:** Make your own copy of another person's project so you can experiment or propose changes.

1. Open a public repository you want to learn from.
2. Click **Fork** near the top-right.
3. Choose your GitHub account as the owner.
4. Keep or edit the repository name.
5. Click **Create fork**.
6. Verify the page now shows your account as the owner and indicates the original repository as its source.

**Fork vs clone:** A fork creates a GitHub-hosted copy under your account. A clone creates a local working copy on your computer. You can clone your fork after forking.

**Practice:** Fork a public documentation repo, make a small improvement in your fork, and learn the pull request workflow. Only submit useful changes that comply with the project's contribution rules.

## 6. Clone a repository

**Goal:** Download a working copy to your computer.

1. Open your repository on GitHub.
2. Click the green **Code** button and choose **HTTPS**.
3. Copy the repository URL. It resembles `https://github.com/YOUR-USERNAME/IT-101-to-Expert-Hands-On-Labs.git`.
4. Open Terminal, PowerShell, or Git Bash on your computer.
5. Run (replace the URL with your actual one):

```bash
git clone https://github.com/YOUR-USERNAME/IT-101-to-Expert-Hands-On-Labs.git
cd IT-101-to-Expert-Hands-On-Labs
git status
```

**Expected:** `git status` says you are on a branch and the working tree is clean. If `git` is not recognized, complete Section 7 first.

## 7. Install Git and configure identity

1. Download Git from https://git-scm.com/downloads and install it for your operating system.
2. Open a terminal and verify:

```bash
git --version
```

3. Set your commit identity (replace with your own values):

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global --list
```

**Explanation:** Git stores the configured name and email in commit metadata. Use an appropriate email address; GitHub provides a private `noreply` email option in account settings.

## 8. Git command workflow

**Goal:** Make a local change and upload it.

From inside your cloned repository:

```bash
mkdir -p linux
printf '# Linux 101\n\nMy Linux practice notes.\n' > linux/README.md
git status
git add linux/README.md
git commit -m "Add Linux training notes"
git push
```

**What each command does:**

| Command | Explanation |
|---|---|
| `mkdir -p linux` | Creates a folder (works in Git Bash/Linux/macOS) |
| `printf ... > linux/README.md` | Creates the Markdown file |
| `git status` | Shows changed, staged, and untracked files |
| `git add ...` | Stages the file for the next commit |
| `git commit -m "..."` | Saves a local version-history checkpoint |
| `git push` | Sends local commits to the remote repository |

In PowerShell, you can create the file using `New-Item -ItemType Directory -Force linux` and `Set-Content linux/README.md '# Linux 101'` instead.

**Verify:** Refresh your GitHub repository and look for `linux/README.md`. GitHub may request browser sign-in, a credential manager, or another supported authentication method. Your GitHub account password is not used as a Git HTTPS password.

### Get changes made by others

```bash
git pull
```

This fetches and integrates remote changes into your current branch. Check `git status` first and commit/stash your own work if necessary.

## 9. Branches and pull requests

**Goal:** Make a change without editing the default branch directly.

1. From the repository folder, make sure your default branch is current:

```bash
git switch main
git pull
git switch -c add-dhcp-lesson
```

If your default branch has another name, use that name instead of `main`.

2. Create or edit `networking/dhcp.md` with your text editor.
3. Stage, commit, and push:

```bash
git add networking/dhcp.md
git commit -m "Add DHCP lesson"
git push -u origin add-dhcp-lesson
```

4. Open your GitHub repository and click **Compare & pull request** (or **Pull requests → New pull request**).
5. Choose base branch `main` and compare branch `add-dhcp-lesson`.
6. Add a title and description of your change.
7. Click **Create pull request**.
8. Review the **Files changed** tab, address feedback, then select **Merge pull request** if permitted.
9. Optionally delete the merged branch using GitHub's button.

**Why:** Branches let you work on a feature separately. A pull request creates a reviewable record before merging.

## 10. Issues and collaboration

**Create an issue:**
1. Open your repo's **Issues** tab (enable Issues in repository settings if absent).
2. Click **New issue**.
3. Title it `Document how DHCP leases work`.
4. Add a description, expected outcome, and checklist.
5. Submit it. Close the issue after the work is completed.

**Collaborators:** In a personal repository, open **Settings → Collaborators** (the exact menu can vary), invite someone by username, and choose appropriate permissions. Avoid granting write access unless needed.

## 11. Undo mistakes safely

### Undo an uncommitted edit to one tracked file

```bash
git restore README.md
```

**Warning:** This discards your uncommitted changes to that file. Check `git diff` first.

### Unstage a file without deleting edits

```bash
git restore --staged README.md
```

### Undo an already committed change safely

```bash
git log --oneline -5
git revert COMMIT_HASH
git push
```

Replace `COMMIT_HASH` with the commit you want to reverse. `git revert` makes a new commit that reverses the earlier change and is usually safer for shared branches than rewriting history.

### Recover a missing commit

```bash
git reflog
```

The reflog can help locate commits previously referenced by your local repository. Ask for help before using destructive reset or force-push commands.

## 12. Delete a repository safely

**Danger: Repository deletion is destructive and may affect collaborators and automation.**

1. Confirm that you are in the correct repository.
2. Download or clone a backup if needed.
3. Open **Settings → General**.
4. Scroll to **Danger Zone**.
5. Choose **Delete this repository**.
6. Read GitHub's warnings and complete the required confirmation steps.

GitHub may offer a limited restoration window, but do not depend on it. **Deleting a repository is different from deleting one file.**

## 13. Security and troubleshooting

### Protect secrets
- Never commit passwords, private keys, API tokens, customer records, or sensitive company information.
- Use a `.gitignore` file for local files that should not be tracked. Example:

```gitignore
.env
*.key
secrets/
```

- `.gitignore` does not remove files already committed; remove tracked secrets, revoke/rotate exposed credentials, and follow a proper history-remediation procedure when necessary.
- Turn on two-factor authentication and use the least privilege needed for collaborators and tokens.

### Common errors

| Error | Likely reason | First action |
|---|---|---|
| `git: command not found` | Git not installed or not on PATH | Install Git and restart terminal |
| `fatal: not a git repository` | Terminal outside a repo | `cd` into cloned repo |
| `nothing to commit` | No new staged changes | Run `git status` |
| Push rejected (`non-fast-forward`) | Remote has commits you lack | `git pull`, resolve conflicts, retry |
| Authentication failed | Credentials/token issue | Reauthenticate with GitHub's supported method |
| Merge conflict | Both sides edited overlapping lines | Open marked file, choose correct text, test, then commit |

### Resolve a simple merge conflict

If Git marks a file with `<<<<<<<`, `=======`, and `>>>>>>>`, open it in a text editor. Decide which lines should remain, remove the conflict markers, save the file, then run:

```bash
git add README.md
git status
```

Complete the merge/rebase as instructed by `git status` (often `git commit` for a merge or `git rebase --continue` for a rebase). Do not blindly remove text you do not understand.

## 14. Practice project and checklist

Create a private practice repository called `github-practice-lab` and complete these tasks:

- [ ] Create a repository with a README
- [ ] Add a `networking/README.md` file in the browser
- [ ] Edit the README and commit
- [ ] Upload a Markdown file from your computer
- [ ] Create a disposable file and delete it
- [ ] Find the deleted file in commit history and restore it
- [ ] Fork a public repository
- [ ] Clone your own repository locally
- [ ] Configure Git name and email
- [ ] Make a local commit and push it
- [ ] Pull remote changes
- [ ] Create a branch and open a pull request
- [ ] Merge the pull request
- [ ] Open and close an issue
- [ ] Practice `git revert` on a harmless commit
- [ ] Enable two-factor authentication and review repository visibility

### Mini capstone: Publish your IT training materials

Use the following repository layout:

```text
IT-101-to-Expert-Hands-On-Labs/
├── README.md
├── github-basics/
│   └── README.md
├── virtual-machines/
│   └── README.md
├── disk-partitioning/
│   └── README.md
└── networking/
    └── README.md
```

Create each folder by adding a file with its full path in GitHub, or use Git locally. Link to the lessons from the root README with relative links, for example:

```markdown
- [GitHub Basics](github-basics/README.md)
- [Virtual Machines](virtual-machines/README.md)
- [Disk Partitioning](disk-partitioning/README.md)
```

**Completion standard:** You can create a repository, make a change in a branch, open and merge a PR, recover a mistaken commit, and explain the difference between fork, clone, pull, and push.

---

**Official references:** [GitHub Docs](https://docs.github.com/en) · [Git Documentation](https://git-scm.com/doc) · [GitHub Skills](https://skills.github.com/)
