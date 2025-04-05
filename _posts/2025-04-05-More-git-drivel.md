# Git Fundamentals: What Your Tutorials Might Have Missed

As I've navigated the sometimes confusing world of Git, I've realized there's a gap between knowing commands and understanding them. While plenty of resources teach you what to type, fewer explain the why behind each command. Let me share some insights that have made Git more than just a mysterious tool I use daily.

## The Reality of `git fetch` vs. `git pull`

Many developers (myself included, initially) use `git pull` as their go-to command for updates:

```bash
# The command everyone knows
git pull
# Sometimes followed by: "Wait, what just happened to my code?"
```

What's often overlooked is that `git pull` is actually two operations in one:

```bash
# What git pull actually does behind the scenes:
git fetch    # First, download changes
git merge    # Then, integrate them immediately
```

This automatic merge can be convenient—but also surprisingly disruptive when you're in the middle of something. A more intentional approach:

```bash
# A more deliberate workflow:
git fetch                 # Get the changes without applying them
git diff origin/main      # See what you're about to integrate
git merge origin/main     # Merge when you're actually ready
```

This small change in workflow has saved me from countless interruptions and merge headaches.

## Why `git push origin HEAD` Is Worth the Extra Typing

Another command worth reconsidering:

```bash
git push
# Sometimes works perfectly, sometimes fails with cryptic messages
```

I've found that `git push origin HEAD` offers more reliability:

```bash
git push origin HEAD   # Explicitly push current branch to matching remote branch
```

This approach is better because:

1. It works consistently regardless of your Git configuration
2. It's clear about what's happening (current branch → same-named remote branch)
3. It avoids confusion when branch tracking isn't set up

The extra few keystrokes have saved me more debugging time than I care to admit.

## Protecting Your Secrets: A Non-Negotiable Practice

One mistake that can have serious consequences: committing API keys or credentials to a public repository.

```bash
# The dangerous habit:
git add .
git commit -m "Finished the API integration"
# Your API keys might now be public
```

A better approach that's become second nature for me:

```bash
# Step 1: Always add .env to .gitignore
echo ".env" >> .gitignore

# Step 2: Create a template for documentation
cp .env .env.example
# (Then remove actual keys from .env.example)

# Step 3: Verify it's working as expected
git status
# .env should NOT appear in the files to be committed
```

If you've already committed sensitive data:

```bash
# Remove from Git tracking but keep the local file
git rm --cached .env

# Check if it's in your history
git log --all -- .env
```

## Properly Cleaning Sensitive Data from Git History

When sensitive data like API keys or credentials has been committed to a Git repository, removing it completely requires careful steps. Here's a detailed guide:

## For Local Repositories (Not Yet Pushed)

If you've only committed sensitive data locally and haven't pushed to a remote repository:

1. **Use interactive rebase to edit history**:
   ```bash
   # Replace N with the number of commits to examine
   git rebase -i HEAD~N
   ```

2. **In the editor that opens, change "pick" to "edit" for commits containing sensitive data**:
   ```
   edit 2c3a8b1 Add API integration with keys
   pick 8d7f235 Update UI components
   ```

3. **When the rebase stops at the commit to edit**:
   ```bash
   # Remove the file from Git but keep it locally
   git rm --cached .env
   
   # Update the commit
   git commit --amend
   
   # Continue the rebase process
   git rebase --continue
   ```

4. **Create a proper .gitignore file** if you haven't already:
   ```bash
   echo ".env" >> .gitignore
   git add .gitignore
   git commit -m "Add .env to .gitignore"
   ```

## For Repositories Already Pushed to Remote

If sensitive data has been pushed to a remote repository, the process is more complex:

### Option 1: Using BFG Repo-Cleaner (Recommended)

BFG is faster and simpler than git filter-branch:

1. **Download BFG Repo-Cleaner** from https://rtyley.github.io/bfg-repo-cleaner/

2. **Create a fresh clone of your repository**:
   ```bash
   # Create a mirror clone
   git clone --mirror git://example.com/my-repo.git
   ```

3. **Run BFG to remove the sensitive file**:
   ```bash
   # Replace .env with your sensitive file
   java -jar bfg.jar --delete-files .env my-repo.git
   ```

4. **Clean up and update references**:
   ```bash
   cd my-repo.git
   git reflog expire --expire=now --all
   git gc --prune=now --aggressive
   ```

5. **Push the cleaned repository**:
   ```bash
   git push --force
   ```

### Option 2: Using git filter-branch

If you can't use BFG:

1. **Remove the file from all commits**:
   ```bash
   git filter-branch --force --index-filter \
     "git rm --cached --ignore-unmatch .env" \
     --prune-empty --tag-name-filter cat -- --all
   ```

2. **Force Git to remove references to old commits**:
   ```bash
   git reflog expire --expire=now --all
   git gc --prune=now --aggressive
   ```

3. **Force push to remote**:
   ```bash
   git push --force
   ```

## Critical Follow-up Steps

After cleaning the repository:

1. **Consider all exposed credentials compromised**:
   - Immediately rotate all API keys, passwords, and tokens that were exposed
   - Generate new credentials for all services

2. **Notify team members**:
   - All collaborators must re-clone the repository or carefully reset their local copies
   - Instructions for collaborators:
     ```bash
     # Fetch the latest state
     git fetch --all
     
     # Reset to match the remote
     git reset --hard origin/main
     
     # OR clone fresh
     git clone https://github.com/user/repo.git new-clone
     ```

3. **Check hosting platform policies**:
   - GitHub, GitLab, and other platforms may cache data
   - Some platforms like GitLab require waiting periods (30+ minutes) after history cleaning
   - Contact support if necessary for complete purging

4. **Implement prevention measures**:
   - Use .gitignore properly
   - Consider pre-commit hooks to catch sensitive patterns
   - Use environment variables or secure credential management tools

Remember: The safest approach is to treat any credentials that have been committed to a repository as compromised, even after cleaning the history. This isn't just good practice—it's essential security hygiene.

## Branching: Your Safety Net for Experimentation

Working directly on the main branch feels efficient until something breaks. I've learned that branching isn't just for teams:

```bash
# Before making significant changes:
git checkout -b feature-new-ui

# Work, commit, test without pressure...

# When confident, push your branch
git push -u origin feature-new-ui
```

This approach has given me the freedom to experiment without the anxiety of breaking production code.

## Commit Messages: Future You Will Thank Present You

I used to write commit messages like:

```
"Fixed bug"
"Updated styling"
"Changes from review"
```

These messages tell almost nothing about what actually changed. Now I aim for:

```bash
# Check what you're about to commit
git status

# Add changes deliberately
git add file1.js file2.js

# Write a message that actually helps later
git commit -m "Fix user session timeout after password change"
```

When I look back at projects months later, these detailed messages have proven invaluable.

## Final Thoughts

Git is powerful but not always intuitive. It's like having a sophisticated tool that can either make your work significantly easier or occasionally more complicated—depending on how you use it.

I've found that taking the time to understand Git commands rather than just memorizing them has made me more confident and prevented many frustrating situations. These practices aren't just about following best practices—they're about making development more predictable and less stressful.

What Git practices have you found most helpful in your development work?

---

*This post reflects practical Git experiences and lessons learned along the way.*
