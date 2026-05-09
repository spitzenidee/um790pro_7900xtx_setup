I have a local gitea instance for a local git repo platform. For some repos I would like to be able to also push to GitHub, e.g. specific branches of tags which I have prepared for public publishing. Is it possible to have a GitHub repo as some kind of "mirror repo" for a local gitea repo and branch (or tag, or something else as discriminator), and if so, what options do I have?

It is entirely possible to synchronize a local Gitea repository with GitHub. Since you require specific discriminators—such as only pushing certain branches or tags—you have several technical approaches ranging from built-in UI features to automated CI/CD workflows.

1. Gitea Native "Push Mirroring"
Gitea includes a built-in Push Mirror feature. This is the most straightforward method but has limitations regarding "discriminators."

Functionality: Gitea automatically pushes all branches, tags, and commits to a target URL (GitHub) whenever the local repository is updated.

Setup: Navigate to Repository Settings > Mirror > Push Mirror Settings. Enter your GitHub repository URL and a Personal Access Token (PAT) for authentication.

Constraint: This generally mirrors the entire repository state. If you need to exclude certain internal branches from GitHub entirely, this native feature might be too broad, as it typically synchronizes all references.

2. Gitea Actions (Recommended for Filtering)
If you only want specific branches (e.g., main or release/*) or tags (e.g., v*) to appear on GitHub, Gitea Actions (built on GitHub Actions syntax) is the most precise tool.

Workflow Example
You can create a .gitea/workflows/mirror.yaml file to handle the logic:

YAML
on:
  push:
    branches:
      - main
    tags:
      - 'v*'

jobs:
  mirror_to_github:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Mirror to GitHub
        run: |
          git push --force --prune https://${{ secrets.GH_PAT }}@github.com/username/repo.git refs/remotes/origin/*:refs/heads/*
Pros: Complete control over triggers. You can ensure "dev" branches never leave your local network while "main" is automatically published.

Cons: Requires a configured Gitea Runner.

3. Server-Side Git Hooks
For a solution that does not require a Runner, you can utilize Gitea's Git Hooks (server-side).

How it works: You modify the post-receive hook on the Gitea server.

Logic: The script checks the branch name being pushed. If the branch matches your public criteria, the script executes a git push to the GitHub remote.

Location: Found in Repository Settings > Git Hooks > Post-Receive.

[!IMPORTANT]
Ensure the Gitea service user has the necessary SSH keys or credentials configured in its environment to authenticate with GitHub.

4. Manual Multi-Remote Setup (Client-Side)
If you prefer manual control over when a "public" version is released, you can configure your local developer machine with multiple remotes.

Add the GitHub remote: git remote add public https://github.com/user/repo.git

Push specific branches: git push public main or git push public --tags

This avoids automation but provides a "human-in-the-loop" check before any code reaches the public internet.

Comparison of Options
Method	Granularity	Complexity	Best For
Push Mirroring	Low (All or nothing)	Low	Full backups/mirrors.
Gitea Actions	High (Branch/Tag filters)	Medium	Automated public releases.
Git Hooks	High (Scriptable)	High	Minimal overhead (no CI/CD).
Multi-Remote	Total (Manual)	Low	Occasional, manual publishing.
Which of these integration levels best fits your current workflow: an automated CI/CD approach or a server-level configuration?

I think gitea actions option is optimal for me. I could configure it such that only branches named with eg. the pattern "github_*" are mirrored. I'm not sure if I also need Tag filtering - what is the use case of tags vs branches, btw?

In the context of Git, the distinction between branches and tags is primarily defined by mutability and intent. Using Gitea Actions to filter by these allows you to strictly control what becomes public on GitHub.

Technical Distinction: Branches vs. Tags
Branches
A branch is a movable pointer to a commit. As you add new commits, the branch pointer automatically advances to the latest one.

State: Dynamic/Mutable.

Primary Use: Ongoing development, feature isolation, or environment-specific code (e.g., your github_* pattern).

Mirroring Logic: Use branches when you want the GitHub repository to reflect a "living" state of your code that will receive future updates.

Tags
A tag is an immutable snapshot of a repository at a specific point in time. Once a tag is created (e.g., v1.0.4), it does not move even if you add new commits to the branch it was created from.

State: Static/Permanent.

Primary Use: Versioning, releases, and milestones.

Mirroring Logic: Use tags when you want to trigger GitHub's "Releases" feature. This ensures that users downloading your code from GitHub receive a verified, stable version rather than a "work-in-progress" branch.

Use Cases for Mirroring
When mirroring to GitHub, the choice between branches and tags usually follows these patterns:

Feature	Branch Mirroring (github_*)	Tag Mirroring (v*)
Visibility	Shows the current progress of public-facing features.	Represents formal software releases.
GitHub UI	Updates the code in the "Code" tab.	populates the "Releases" and "Packages" sections.
Automation	Can trigger GitHub Actions on the remote side for CI.	Useful for triggering production deployment pipelines.
Stability	May contain bugs (depending on your local workflow).	Should ideally represent "vetted" or "production-ready" code.
Implementing the Pattern in Gitea Actions
To implement your specific requirement where only branches starting with github_ are mirrored, you can use the following YAML configuration. This setup also includes an optional section for tags if you decide to use them later.

YAML
name: Mirror to GitHub

on:
  push:
    # Filter for branch patterns
    branches:
      - 'github_*'
    # Filter for tag patterns (optional)
    tags:
      - 'v*'

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Push to GitHub
        env:
          GITHUB_TOKEN: ${{ secrets.GH_PAT }}
        run: |
          # Determine the ref type (branch or tag)
          REF_NAME=${GITHUB_REF#refs/*/}
          
          # Push the specific ref that triggered the action
          git push --force https://your-username:$GITHUB_TOKEN@github.com/your-username/your-repo.git $GITHUB_REF:refs/heads/$REF_NAME
Key Considerations for your Pattern:
Naming Mapping: If you push a branch named github_main to GitHub, it will appear there as github_main. If you want it to appear as main on GitHub, you would need to adjust the push command to: git push ... refs/heads/github_main:refs/heads/main.

Secret Management: Store your GitHub Personal Access Token (PAT) in Gitea's Settings > Secrets to avoid exposing credentials in the YAML file.

Atomic Pushes: Using GITHUB_REF ensures that only the specific branch or tag that was pushed to Gitea is mirrored, rather than trying to sync the entire repository state.

Does your current workflow involve specific version numbers, or is your public publishing primarily focused on sharing the latest state of certain branches?

Mhmmm. Okay. My idea is: I develop locally only, until I judge the state of development being "ready for the public". Then I could either create a new branch named eg. "github_v2.5" from the repo state OR a snapshot a similar tag.



I would likely try to establish the following best practice:

1. Develop locally with gitea

2. At some point: code is ready for public GitHub => create branch (or tag?) named "github_v2.5"

3. Code state is mirrored to GitHub

4. If I find bugs: fix locally in another branch

5. When fix is validated: merge to "github_v2.5" branch or tag

6. Delete local bugfix branch

7. For new release or feature: create new branch or tag named "github_v3.0"



Would you rate this flow/best practices as sensible and fitting or do you propose changes? And: are branches or tags the way to go?

Your proposed workflow is logical, but it treats branches in a way that might lead to "branch clutter" on GitHub over time. In the Git ecosystem, there is a standard distinction between the stream of development (branches) and points in time (tags).

Here is an evaluation of your flow and a recommended optimization to keep your public repository clean.

1. Evaluation of your Proposed Flow
Your idea of isolating public releases is sound. However, creating a new branch for every version (e.g., github_v2.5, github_v3.0) is unconventional.

The Issue: On GitHub, users usually expect one primary branch (like main) that always contains the latest stable code. If you use version-named branches, users have to manually switch branches to find the latest version, and your "default" branch will quickly become outdated.

The Bugfix Logic: You mentioned merging fixes into a tag. Technically, you cannot merge into a tag. A tag is a static pointer. To "update" a tag, you would have to delete and recreate it, which is considered bad practice in collaborative environments (it breaks things for anyone who already pulled that tag).

2. The Optimized "Best Practice" Flow
Instead of creating a new branch for every version, use a persistent public branch combined with tags.

The Setup
Local Development: Work on your internal branches (e.g., main, feature-x) in Gitea.

The Mirror Branch: Create one persistent branch in Gitea named production or public. Configure your Gitea Action to mirror only this branch to GitHub's main branch.

The Workflow
Release Ready: Merge your local main into the public branch.

Tagging: Create a tag (e.g., v2.5) on the public branch in Gitea.

The Push: Push both the branch and the tag to Gitea.

The Mirror: Your Gitea Action pushes the public branch to GitHub (as main) and also pushes the tag.

Bugfixes: * Fix the bug locally.

Merge the fix into your local main AND the public branch.

Create a new "patch" tag (e.g., v2.5.1).

Push to Gitea; GitHub updates automatically.

3. Branches vs. Tags: Which to use?
You should use both, but for different purposes:

Feature	Branch (The "Channel")	Tag (The "Snapshot")
Identity	A moving target (e.g., main).	A fixed point (e.g., v1.0).
GitHub UI	Determines what people see first.	populates the "Releases" tab.
Purpose	Shows the latest public state.	Provides a downloadable version that never changes.
Logic	"I want to see the current code."	"I want to download exactly version 2.5."
4. Updated Gitea Action for this Flow
This configuration maps your local public branch to the GitHub main branch and handles tags correctly.

YAML
on:
  push:
    branches:
      - public
    tags:
      - 'v*'

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Mirror
        run: |
          # Push the public branch to GitHub's main
          git push --force https://${{ secrets.GH_PAT }}@github.com/user/repo.git public:main
          # Push all tags
          git push --tags https://${{ secrets.GH_PAT }}@github.com/user/repo.git
Why this is better:
Cleanliness: Your GitHub repository only has one branch (main), making it easy for others to clone and contribute.

History: The Releases section on GitHub will be automatically populated by your tags, providing a clear history of your progress.

Automation: You don't have to change your YAML file every time you bump a version; it just works based on the v* pattern.

Does the concept of a single "public" mirror branch feel more manageable than creating new branches for every version?

Sounds good, few questions:

- can I also push to the local "public" branch frequently but not create a tag all the time?

- I don't intend to push to GitHub "main" directly, this branch should be protected and only be updated via PRs

- shouldn't I better create tags on GitHub only, or are tags "cross-branch" in the end? Do I create a tag from a specific branch...?

Yes, those are valid technical considerations. Here is the breakdown of how Git handles these scenarios and how to adjust your workflow for GitHub's protected branches.

1. Frequent Pushes vs. Tags
You can push to your local public branch as often as you like without creating a tag.

Branches are for continuous updates. Every time you push to the public branch, Gitea Actions will trigger and update the corresponding branch on GitHub.

Tags are optional markers. You only create them when you want to signal a specific milestone (like a release).

Workflow: You might push 10 times to public to fix typos or small items, and only on the 11th push do you decide to "tag" it as v1.1.0.

2. Mirroring to a Protected Branch
If GitHub's main branch is protected and requires a Pull Request (PR), a standard git push --force from Gitea Actions will be rejected by GitHub unless the credentials used have "Bypass branch protection" permissions.

If you want to maintain the PR requirement on GitHub, you should change the destination of your mirror:

The "Staging" Approach: Instead of Gitea public -> GitHub main, configure the action to push Gitea public -> GitHub proposals (or any other name).

The PR Step: On GitHub, you then manually open a PR from proposals to main. This allows you to use GitHub’s code review, automated testing, or status checks before the code actually hits your public main branch.

Modified Push Command in YAML:

YAML
run: |
  # Push local 'public' to a specific 'incoming' branch on GitHub
  git push --force https://${{ secrets.GH_PAT }}@github.com/user/repo.git public:refs/heads/updates-from-local
3. Understanding Tags: Scope and Creation
Tags are often misunderstood as being "inside" a branch, but technically they are independent objects.

Are tags "cross-branch"?
Yes. A tag points to a commit hash, not a branch name.

If Commit A is part of both your develop branch and your public branch, a tag on Commit A effectively exists "on" both branches.

However, in the GitHub UI, a tag will often be associated with the branch it was first seen on, or the default branch.

Should you create tags on GitHub or Gitea?
You should create them locally/Gitea.

Reason: Git is a distributed system. If you create a tag on GitHub but not in your local Gitea instance, your local environment will be "out of sync" with the official release versions.

Best Practice: Create the tag in Gitea. When Gitea Actions pushes to GitHub using git push --tags, the tags will appear on GitHub automatically, perfectly aligned with the correct commits.

Do I create a tag from a specific branch?
Yes, typically. When you run git tag v1.0, Git tags whatever commit you are currently "on" (the HEAD).

Comparison: PR Workflow vs. Direct Mirror
Feature	Direct Mirror (Bypass Protection)	PR Workflow (Staging Branch)
Effort	Zero (fully automated).	Low (requires manual click on GitHub).
Control	Gitea is the "Source of Truth".	GitHub acts as a final "Gatekeeper".
GitHub Features	Uses GitHub only for hosting.	Uses GitHub for Reviews/Discussions/CI.
Security	Requires admin-level Token.	Works with standard write permissions.
Given that you want to use PRs on GitHub, would you prefer the mirror to create a new branch for every update, or simply keep one single "staging" branch on GitHub that you merge into main repeatedly?

For tag creation: I expect my local repos and releases to be potentially different from public ones on GitHub (e.g. specific "plugins" will be missing from the public branch, but not the "internal" ones). This means: If I create a tag, this must never trigger that "internal code" is mirrored onto GitHub. Is this already ensured with the current proposal or are there things to change?

The current proposal requires a specific adjustment to prevent "internal" code leakage. In Git, a tag is a pointer to a specific commit. If you tag a commit on an internal branch and push that tag to GitHub, Git will also push all the code (the entire file tree) associated with that commit, even if the branch itself isn't pushed.

To ensure your internal plugins and private code never reach GitHub, you must implement a structural separation and a filtered Action.

1. The "Leakage" Risk
If your local main contains internal plugins and your local public does not:

Safe: You tag a commit on the public branch. Only public code is pushed.

Dangerous: You are accidentally checked out on main, you create tag v2.0, and push. The Action triggers, sees a tag starting with v, and pushes it to GitHub. GitHub now has the v2.0 tag and all the internal code contained in that commit.

2. Solution: The "Double-Lock" Gitea Action
You can configure your Gitea Action to be more restrictive. Instead of pushing any tag that matches a pattern, the Action should only push tags that are ancestors of your public branch.

Optimized YAML Logic
YAML
on:
  push:
    branches:
      - public
    tags:
      - 'v*'

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Essential to see the history and verify tag-branch relationship

      - name: Selective Mirror
        run: |
          # 1. Update the 'staging' branch on GitHub with the public branch content
          git push --force https://${{ secrets.GH_PAT }}@github.com/user/repo.git public:refs/heads/updates-from-local

          # 2. Check if the current trigger is a tag
          if [[ $GITHUB_REF == refs/tags/* ]]; then
            TAG_NAME=${GITHUB_REF#refs/tags/}
            
            # 3. Verify if the tag exists within the 'public' branch history
            if git merge-base --is-ancestor $TAG_NAME public; then
              echo "Tag $TAG_NAME is on the public branch. Mirroring to GitHub..."
              git push https://${{ secrets.GH_PAT }}@github.com/user/repo.git $TAG_NAME
            else
              echo "Abort: Tag $TAG_NAME is NOT on the public branch. Skipping to prevent code leak."
              exit 0
            fi
          fi
3. Managing the Code Difference (Internal vs. Public)
Since you want the public branch to be a "subset" of your local code (missing plugins), you have two primary methods to manage this:

Option A: The "Sanitized" Branch (Manual/Scripted)
You maintain a public branch that is never directly merged from main. Instead, you "cherry-pick" features from main to public, or you have a script that deletes the internal plugin folders before committing to public.

Option B: Git Submodules (Recommended for Plugins)
If your internal plugins are in separate directories:

Keep the plugins in their own private Gitea repositories.

Add them as Submodules to your local Gitea main branch.

In the public branch, simply do not include the .gitmodules file or the plugin folders.

Result: When you mirror the public branch, the plugins physically don't exist in that branch's file tree, making a leak impossible.

Summary of Best Practices for your Case
Never use git push --tags: This command pushes every tag in your local DB. Always push specific tags or use the filtered Action logic above.

The "Ancestor" Check: The git merge-base --is-ancestor command is your fail-safe. It programmatically ensures that a tag only goes to GitHub if the code it points to is already part of the public history.

Local Validation: Before pushing to GitHub, you can run git ls-tree -r public locally to see exactly which files are in the public branch.

Does the "Ancestor Check" in the Action provide enough security for your workflow, or do you need a more automated way to "strip" the internal plugins during the mirror process?

Can this gitea action be made transportable, meaning: I can easily drop it into any repo, and such a repo only has to satisfy some specific requirements, such as having a "gitea_publish_action.cfg" in the repo root (or somewhere else) whichl defines target public GitHub repo URL and potentially some other sensible variables?

To make the workflow transportable, you can design the YAML to act as an engine that consumes a configuration file. This allows you to copy the same .gitea/workflows/mirror.yaml into any repository without modification, as long as the repository contains a mirror_config.env file and the necessary secret.

1. The Configuration File (mirror_config.env)
Place this file in your repository root. It defines the specific behavior for that repository.

Bash
# Target GitHub repository (HTTPS URL without the protocol)
TARGET_REPO="github.com/username/public-repo-name.git"

# The local branch that is allowed to be mirrored
SOURCE_BRANCH="public"

# The destination branch name on GitHub
DEST_BRANCH="main"

# The tag pattern to allow
TAG_FILTER="v*"
2. The Transportable Gitea Action
This workflow script uses standard shell commands to parse the configuration file at runtime.

YAML
name: Transportable Mirror Action

on:
  push:
    branches:
      - '**' # Triggers on all, logic below filters
    tags:
      - '**'

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Load Config and Mirror
        env:
          # The only repo-specific requirement in Gitea UI
          GH_TOKEN: ${{ secrets.GH_PAT }} 
        run: |
          CONFIG_FILE="mirror_config.env"

          if [ ! -f "$CONFIG_FILE" ]; then
            echo "Error: $CONFIG_FILE not found. Skipping mirror."
            exit 0
          fi

          # Parse variables from config
          source $CONFIG_FILE

          # Determine current ref
          REF_NAME=${GITHUB_REF#refs/*/}

          # LOGIC A: Branch Mirroring
          if [[ $GITHUB_REF == refs/heads/* ]]; then
            if [ "$REF_NAME" == "$SOURCE_BRANCH" ]; then
              echo "Mirroring branch $SOURCE_BRANCH to $DEST_BRANCH..."
              git push --force "https://x-access-token:${GH_TOKEN}@${TARGET_REPO}" "$SOURCE_BRANCH:refs/heads/$DEST_BRANCH"
            else
              echo "Branch $REF_NAME does not match $SOURCE_BRANCH. Skipping."
            fi
          fi

          # LOGIC B: Tag Mirroring (with Ancestor Check)
          if [[ $GITHUB_REF == refs/tags/* ]]; then
            if [[ $REF_NAME == $TAG_FILTER ]]; then
              if git merge-base --is-ancestor "$REF_NAME" "$SOURCE_BRANCH"; then
                echo "Tag $REF_NAME is on $SOURCE_BRANCH. Mirroring..."
                git push "https://x-access-token:${GH_TOKEN}@${TARGET_REPO}" "$REF_NAME"
              else
                echo "Abort: Tag $REF_NAME is not an ancestor of $SOURCE_BRANCH. Potential leak blocked."
              fi
            fi
          fi
Requirements for the "Drop-in" to work
For this to remain transportable and secure, each repository must satisfy these three conditions:

Configuration File: A file named mirror_config.env must exist in the root with the variables defined above.

Naming Consistency: The local branch intended for public consumption must match the SOURCE_BRANCH variable in the config.

Gitea Secret: You must add a Secret named GH_PAT to the Gitea repository (or organization) settings. This token needs repo scope on GitHub.

Why this structure is efficient
Decoupled Logic: The YAML handles the "how" (Git commands, ancestor checks, security), while the .env file handles the "where" (URLs, branch names).

Leak Prevention: The git merge-base check is hardcoded into the workflow logic, ensuring that even if someone creates a tag on an internal branch, the transportable action will refuse to push it.

Authentication: Using x-access-token with the HTTPS URL is the standard way to authenticate Git over HTTPS in automated environments.

Do you plan to manage these configuration files manually for each repo, or do you have an automated way of generating them when a new repo is created?

For the gitea secret holding the GitHub PAT: do I understand correctly that it's a configuration inside the gitea installation/instance and not somewhere "in plain sight" inside of any repo? I just want to make clear to me if there is any risk of unintended PAT leakage via the autonomously running gitea action?

Your understanding is correct. Gitea Secrets are designed specifically to keep sensitive information like Personal Access Tokens (PATs) out of the repository's codebase and version history.

1. Storage and Visibility
Gitea Secrets are not stored within the repository files. They are stored in Gitea's internal database.

UI Visibility: Once you enter a secret in Settings > Actions > Secrets, it becomes "Write-Only." You (and other users) can delete or update it, but the actual value can never be viewed again through the web interface.

Repository Visibility: The secret does not exist in your .git history. If someone clones your repository, they do not get your secrets.

Injection: The secret only becomes available to the Gitea Runner (the machine executing the action) at the exact moment the job starts. It is injected as an environment variable into the runner's temporary environment.

2. Risk of Leakage via Actions
While the storage is secure, there are operational risks to consider during the execution of an Action.

Log Masking
Gitea Actions (like GitHub Actions) attempts to mask secrets. If your script accidentally prints the secret to the standard output (e.g., echo $GH_TOKEN), the Gitea log will usually show *** instead of the actual token.

[!CAUTION]
Warning: Do not rely solely on log masking. Certain commands (like git -v or debug modes) might output secrets in formats the masker does not recognize.

Malicious YAML Modification
If an unauthorized person gains the ability to commit to your repository, they could modify the .gitea/workflows/mirror.yaml file to "exfiltrate" the secret (e.g., sending it to an external server using curl).

Mitigation: Use Gitea's Protected Branches feature for your main development branches to ensure only authorized users can modify the workflow files.

Runner Security
If the machine running your Gitea Runner is compromised, an attacker could potentially inspect the environment variables of running processes to capture the token.

Mitigation: Ensure your Gitea Runner is hosted on a secure, private server (or a trusted Docker environment) with restricted access.

3. Security Best Practices for PATs
To minimize the impact of a potential leakage:

Fine-Grained Scopes: On GitHub, do not use a "Classic" PAT with full repo access if possible. Use a Fine-Grained Personal Access Token that is restricted only to the specific repository and only has "Contents: Write" permissions.

Short Expiration: Set an expiration date for your PAT and rotate it periodically.

Dedicated "Bot" Account: Consider creating a separate GitHub account (a "machine user") specifically for this mirroring task. This keeps your personal GitHub account separate from the automation.

Use x-access-token: In your Git commands, use the standard x-access-token format as shown in the previous example. This is a recognized pattern for temporary tokens.

Bash
git push https://x-access-token:${GH_TOKEN}@github.com/user/repo.git
By following this approach, the token remains a "black box" that Gitea provides to the script only when needed, keeping it entirely separate from your "plain sight" repository content.

Does this clarify the security boundary for you, or would you like to know more about how Gitea handles secret encryption in its database?

So it would make sense to ALWAYS have a .gitignore alongside the gitea action yaml which excludes the yaml from being pushed to the public GitHub remote?

Actually, using .gitignore for this purpose is a common misconception. In Git, .gitignore only prevents untracked files from being added to the repository. Since your Gitea Action YAML and configuration file must be tracked (committed) for Gitea to see and run them, adding them to .gitignore will have no effect on their visibility or their inclusion in a git push.

If you want your public GitHub repository to be "clean" (without the .gitea folder or the .env config), you need to handle this via the logic in your script.

1. Why .gitignore is not the solution
Tracked vs. Untracked: If a file is already in your repository history, .gitignore is ignored by Git for that file.

Push Logic: When you run git push, you are pushing commits, not just files. If a commit contains the .gitea folder, that folder will arrive at the destination.

2. The "Sanitized Push" Strategy
To keep your GitHub mirror free of Gitea-specific automation files, you should modify your Gitea Action to "scrub" the files before the push happens on the runner.

Updated Transportable Logic
You can add a "Cleanup" step to your script. Since the Action runs on a temporary runner, you can delete files and create a "synthetic" commit that only exists for the duration of the push to GitHub.

YAML
      - name: Load Config and Mirror
        run: |
          source mirror_config.env
          
          # 1. Strip internal automation files from the runner's workspace
          rm -rf .gitea/
          rm mirror_config.env
          
          # 2. Create a temporary 'clean' state
          git config user.name "Gitea Mirror Bot"
          git config user.email "bot@gitea.local"
          git add -A
          git commit -m "Public release cleanup" --allow-empty

          # 3. Push this specific 'clean' commit to GitHub
          # Note: We push the current HEAD to the destination branch
          git push --force "https://x-access-token:${GH_TOKEN}@${TARGET_REPO}" HEAD:refs/heads/$DEST_BRANCH
3. Evaluation of the "Clean" Approach
Approach	Visibility on GitHub	Complexity	Recommendation
Direct Push	Gitea YAML and .env are visible.	Low	Best if you don't care about "messy" repos.
Sanitized Push	GitHub repo is 100% clean code.	Medium	Best for professional/public-facing projects.
.gitignore	No change (files still visible).	Low	Do not use for this purpose.
Is it necessary to hide them?
Security: As long as your secrets are in Gitea Secrets and not in the .env file, there is no security risk in the YAML being public.

Aesthetics: Most developers prefer a clean public repo. Seeing a .gitea folder on GitHub can be confusing for users who might try to use those actions on GitHub (where they will fail or behave differently).

Summary
You should not use .gitignore. Instead, decide if you want a "transparent" mirror (where the automation is visible) or a "sanitized" mirror. If you want it clean, use the rm -rf and git commit approach within the Action script itself to prune the files before they ever reach GitHub.

Does your public-facing code contain any other "internal" files or documentation that you would also want to prune using this method?

FOLLOW-UP QUESTION:
Does your public-facing code contain any other "internal" files or documentation that you would also want to prune using this method?

I want a sanitized mirror always. Can this sanitation also be made configurable or does it add too high of complexity? I mean:



In repo A I want to sanitize the gitea yaml but also a specific dockerfile, some config or "plugin code".



In repo B I want to sanitize the gitea yaml but then the directory ".github/skills".



And repo C: sanitize the gitea yaml and nothing else.

Making this configurable is actually quite straightforward and doesn't add much complexity to the script. We just need to treat your configuration file as a "shopping list" of items to delete before the code hits GitHub.

By adding a single variable to your .env file, you can customize the "sanitation level" for every repository.

1. Updated Configuration File (mirror_config.env)
In each repo, you simply list the files or directories you want to vanish.

Bash
# Target and Branch settings
TARGET_REPO="github.com/username/public-repo-name.git"
SOURCE_BRANCH="public"
DEST_BRANCH="main"

# SANITATION LIST: Space-separated paths to remove
# Note: Always include '.gitea' and 'mirror_config.env' themselves!
EXCLUDE_LIST=".gitea/ mirror_config.env Dockerfile.internal internal_plugins/ .github/skills/"
2. The Enhanced Transportable Action
The logic now iterates through your EXCLUDE_LIST. It uses a "soft" delete (rm -rf) so the script won't crash if a file listed in the config is missing from a specific branch.

YAML
name: Universal Sanitized Mirror

on:
  push:
    branches: [ "public" ]
    tags: [ "v*" ]

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Sanitize and Mirror
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}
        run: |
          CONFIG_FILE="mirror_config.env"
          [ ! -f "$CONFIG_FILE" ] && echo "No config found." && exit 0
          
          source "$CONFIG_FILE"

          # 1. THE SANITATION LOOP
          echo "Starting sanitation..."
          for item in $EXCLUDE_LIST; do
            if [ -e "$item" ]; then
              echo "Removing: $item"
              rm -rf "$item"
            fi
          done

          # 2. CREATE THE TEMPORARY CLEAN COMMIT
          git config user.name "Mirror Bot"
          git config user.email "bot@gitea.local"
          git add -A
          
          # Only commit if there are actually changes to mirror
          if ! git diff-index --quiet HEAD; then
            git commit -m "chore: sanitized public release"
          else
            echo "No changes to commit after sanitation."
          fi

          # 3. PUSH LOGIC
          REF_NAME=${GITHUB_REF#refs/*/}
          if [[ $GITHUB_REF == refs/heads/* ]]; then
            # Push the NEW clean commit we just made (HEAD) to GitHub
            git push --force "https://x-access-token:${GH_TOKEN}@${TARGET_REPO}" HEAD:refs/heads/$DEST_BRANCH
          
          elif [[ $GITHUB_REF == refs/tags/* ]]; then
            # For tags, we push the tag itself. 
            # Note: The tag still points to the ORIGINAL (unsanitized) commit.
            # If you need tags to point to sanitized code, that's a higher level of complexity.
            git push "https://x-access-token:${GH_TOKEN}@${TARGET_REPO}" "$REF_NAME"
          fi
3. Important Technical Nuance: The Tag Problem
There is one "gotcha" with your requirement for tags:

Branches: The script above works perfectly for branches. It creates a new, clean commit and pushes that to GitHub.

Tags: In Git, a tag is a permanent link to a specific commit hash.

If you tag a commit in Gitea, that commit contains the internal files.

When the Action pushes that tag to GitHub, Git pushes the commit the tag points to.

Result: Even if your main branch on GitHub is clean, someone could switch to the v2.5 tag and see all your internal plugins.

The Solution for "Sanitized Tags"
If you strictly need the tags on GitHub to be clean too, the Action must:

Check out the public branch.

Perform the sanitation (delete files).

Commit.

Create a new tag on that specific clean commit.

Push that new tag to GitHub.

4. Complexity Rating
Is this too complex? No. It adds about 5 lines of shell script. The benefit is that your Gitea instance remains your "Master Studio" where everything (internal and external) exists, while your GitHub profile remains a polished, professional storefront.

One final check: For your tags, is it acceptable if the tag on GitHub points to the "internal" version of the code (since most people just look at the main branch), or is it a hard requirement that the tags are also "scrubbed" of all plugin code?

FOLLOW-UP QUESTION:
For your tags, is it a hard requirement that the tagged code on GitHub is also "scrubbed" of all internal plugin code?

Should and can we fully refrain from including tags in the gitea action? I only work and publish branch based, and tagging I will do manually on GitHub. Tag drift between local gitea and public GitHub is no showstopper (should this happen)

You can absolutely refrain from mirroring tags. This simplifies the workflow significantly and eliminates the primary "security leak" vector where internal code could accidentally reach the public via a tag pointer.

1. The "Branch-Only" Strategy
By removing tags from the automation, you establish a clear boundary: Gitea handles the logic and sanitation, while GitHub handles the official presentation.

Why it works: GitHub allows you to create tags and releases directly through its web interface on any commit. Since your Action pushes a "clean" commit to the GitHub branch, you can simply go to GitHub and tag that specific commit.

Security Benefit: There is zero risk of an "internal" tag from your Gitea instance accidentally pushing private code to GitHub.

Tag Drift: As you noted, this is a non-issue. Your Gitea tags remain your private internal milestones, and your GitHub tags remain your public version history.

2. Updated "Branch-Only" Transportable Action
This version removes all tag logic. It only listens for pushes to your public branch, runs the sanitation, and updates GitHub.

YAML
name: Sanitized Branch Mirror

on:
  push:
    branches:
      - 'public' # Or whatever your source branch is

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Sanitize and Mirror
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}
        run: |
          CONFIG_FILE="mirror_config.env"
          [ ! -f "$CONFIG_FILE" ] && echo "Config missing, skipping." && exit 0
          
          source "$CONFIG_FILE"

          # 1. SCRUBBING
          for item in $EXCLUDE_LIST; do
            [ -e "$item" ] && rm -rf "$item"
          done

          # 2. PREPARE CLEAN STATE
          git config user.name "Gitea Mirror"
          git config user.email "mirror@local.gitea"
          git add -A
          
          # Only proceed if there is new code to push
          if ! git diff-index --quiet HEAD; then
            git commit -m "chore: public release synchronization"
            
            # 3. PUSH TO GITHUB
            # We push the temporary 'clean' HEAD to the remote destination branch
            git push --force "https://x-access-token:${GH_TOKEN}@${TARGET_REPO}" HEAD:refs/heads/$DEST_BRANCH
          else
            echo "No changes after sanitation. Nothing to mirror."
          fi
3. Simplified Configuration (mirror_config.env)
You can now strip the tag-related variables from your config file for a leaner setup.

Bash
TARGET_REPO="github.com/username/repo-name.git"
SOURCE_BRANCH="public"
DEST_BRANCH="main"
EXCLUDE_LIST=".gitea/ mirror_config.env internal_configs/ private_plugins/"
4. Your Manual Workflow on GitHub
Once this Action finishes running:

Open your GitHub repository in the browser.

Go to the Tags or Releases section.

Click "Create a new release".

Choose the main branch (which now contains your sanitized code).

Type in your version (e.g., v1.0.0).

This manual step acts as a final "human-in-the-loop" quality check. You are verifying that the code on GitHub looks correct before formally labeling it as a release.

Comparison: Automated vs. Manual Tagging
Feature	Automated (Mirrored)	Manual (GitHub Only)
Complexity	Higher (Needs ancestor checks).	Lower (Action is very simple).
Safety	Risk of leaking private history.	Impossible to leak via tags.
Sync	Local and Remote match.	Local and Remote may differ.
Effort	Zero manual steps.	One click per release on GitHub.
This "Branch-Only" approach is arguably the most robust way to handle a hybrid public/private setup. It keeps the automation simple and the security tight.

Does this simplified logic cover all the edge cases for your multi-repo setup?

I think so! :) can please give me the final state as a step by step task plan including all necessary configs, paths, ui options and so in, please? I'd like it in markdown suitable for usage as a "readme"-like doc for my local process documentation

This documentation provides a standardized, transportable process for mirroring local Gitea repositories to GitHub while ensuring internal code, configuration, and automation files are "scrubbed" from the public version.

Gitea to GitHub: Sanitized Mirroring Process
1. Prerequisites & Preparation
GitHub Setup
Create Repository: Ensure a target repository exists on GitHub.

Generate PAT: * Go to GitHub Settings > Developer Settings > Personal Access Tokens (Fine-grained).

Grant Read and Write access to "Contents" for the specific repository.

Copy the token; you will not see it again.

Gitea Global Setup
Enable Actions: Ensure Gitea Actions is enabled in your app.ini and a Runner is registered.

Store Secret: * In your Gitea repo, navigate to Settings > Actions > Secrets.

Add a new secret named GH_PAT.

Paste your GitHub token as the value.

2. Repository Configuration
Each repository requires two files to enable the mirroring engine.

File A: The Configuration (mirror_config.env)
Place this file in the root of your Gitea repository. It defines what goes where and what stays private.

Bash
# target-repo.git (without https://)
TARGET_REPO="github.com/your-username/public-repo-name.git"

# The local branch to mirror
SOURCE_BRANCH="public"

# The branch name on GitHub
DEST_BRANCH="main"

# List of files/folders to delete before pushing to GitHub
# ALWAYS include .gitea/ and mirror_config.env
EXCLUDE_LIST=".gitea/ mirror_config.env internal_configs/ private_plugins/ Dockerfile.dev"
File B: The Mirror Engine (.gitea/workflows/mirror.yaml)
This is the transportable logic. You do not need to edit this file between different repositories.

YAML
name: Sanitized Mirror Engine

on:
  push:
    branches:
      - 'public' # Triggers only when the public branch is updated

jobs:
  mirror-to-github:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Sanitize and Push
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}
        run: |
          CONFIG_FILE="mirror_config.env"
          if [ ! -f "$CONFIG_FILE" ]; then
            echo "Config missing. Action aborted."
            exit 0
          fi
          
          source "$CONFIG_FILE"

          # 1. Sanitation: Remove internal files
          echo "Sanitizing workspace..."
          for item in $EXCLUDE_LIST; do
            if [ -e "$item" ]; then
              rm -rf "$item"
              echo "Removed: $item"
            fi
          done

          # 2. Commit Clean State
          git config user.name "Mirror Bot"
          git config user.email "mirror@gitea.local"
          git add -A
          
          if ! git diff-index --quiet HEAD; then
            git commit -m "chore: sanitized public synchronization"
            
            # 3. Push to GitHub
            echo "Pushing to GitHub..."
            git push --force "https://x-access-token:${GH_TOKEN}@${TARGET_REPO}" HEAD:refs/heads/$DEST_BRANCH
          else
            echo "No changes detected after sanitation."
          fi
3. The Publication Workflow (Step-by-Step)
Follow these steps whenever you are ready to publish code to the public:

Develop Locally: Work on your internal branches (e.g., main or feature-x).

Prepare for Public: Merge the desired state into your local public branch.

git checkout public

git merge main

Validate Config: Ensure mirror_config.env contains all files that should not appear on GitHub.

Push to Gitea: * git push origin public

Automation: Gitea Actions will automatically:

Clone the repo on the runner.

Delete everything in the EXCLUDE_LIST.

Commit the "clean" state.

Force-push to the GitHub main branch.

GitHub Tagging (Manual):

Go to GitHub.com.

Navigate to Releases > Draft a new release.

Select the main branch and create your version tag (e.g., v1.0.0).

4. Security & Troubleshooting
Secret Masking: Gitea will mask the GH_PAT in logs, but avoid using set -x in the shell script to prevent accidental leaks of the parsed token.

Branch Protection: If the GitHub main branch is protected, ensure the GH_PAT belongs to an account with "Bypass branch protection" rules, or change DEST_BRANCH to a staging branch (e.g., incoming) and use PRs on GitHub.

Exclusion Verification: To test what will be sent to GitHub without actually pushing, you can run the rm -rf loop locally on a temporary branch and use git status to see the resulting file tree.

This setup provides a high-security, low-maintenance "firewall" between your local development environment and your public GitHub presence.

Does this structure meet your documentation needs, or should we add a section on how to handle submodules if you ever use them?

Can you create this as an attachment in markdown I can download from this chat?

File B: The Mirror Engine (.gitea/workflows/mirror.yaml)
This is the transportable logic. You do not need to edit this file between different repositories.

YAML
name: Sanitized Mirror Engine

on:
  push:
    branches:
      - 'public' # Triggers only when the public branch is updated

jobs:
  mirror-to-github:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Sanitize and Push
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}
        run: |
          CONFIG_FILE="mirror_config.env"
          if [ ! -f "$CONFIG_FILE" ]; then
            echo "Config missing. Action aborted."
            exit 0
          fi
          
          source "$CONFIG_FILE"

          # 1. Sanitation: Remove internal files
          echo "Sanitizing workspace..."
          for item in $EXCLUDE_LIST; do
            if [ -e "$item" ]; then
              rm -rf "$item"
              echo "Removed: $item"
            fi
          done

          # 2. Commit Clean State
          git config user.name "Mirror Bot"
          git config user.email "mirror@gitea.local"
          git add -A
          
          if ! git diff-index --quiet HEAD; then
            git commit -m "chore: sanitized public synchronization"
            
            # 3. Push to GitHub
            echo "Pushing to GitHub..."
            git push --force "https://x-access-token:${GH_TOKEN}@${TARGET_REPO}" HEAD:refs/heads/$DEST_BRANCH
          else
            echo "No changes detected after sanitation."
          fi
3. The Publication Workflow (Step-by-Step)
Follow these steps whenever you are ready to publish code to the public:

Develop Locally: Work on your internal branches (e.g., main or feature-x).

Prepare for Public: Merge the desired state into your local public branch.

git checkout public

git merge main

Validate Config: Ensure mirror_config.env contains all files that should not appear on GitHub.

Push to Gitea:

git push origin public

Automation: Gitea Actions will automatically:

Clone the repo on the runner.

Delete everything in the EXCLUDE_LIST.

Commit the "clean" state.

Force-push to the GitHub main branch.

GitHub Tagging (Manual):

Go to GitHub.com.

Navigate to Releases > Draft a new release.

Select the main branch and create your version tag (e.g., v1.0.0).

4. Security & Troubleshooting
Secret Masking: Gitea will mask the GH_PAT in logs.

Branch Protection: If the GitHub main branch is protected, ensure the GH_PAT belongs to an account with "Bypass branch protection" rules.

Exclusion Verification: To test what will be sent to GitHub without actually pushing, you can run the rm -rf loop locally on a temporary branch and use git ls-tree -r HEAD to see the resulting file tree.
"""

with open("gitea-github-mirror-guide.md", "w") as f:
f.write(markdown_content)

Your Markdown file is ready for download. It contains the finalized step-by-step task plan, the configuration file structure, and the reusable Gitea Action YAML logic.

[file-tag: code-generated-file-0-1778175917176961815]

### Quick Summary of the Documentation:
* **Setup:** Use Gitea Secrets for the GitHub PAT to ensure no credentials are in your code.
* **Configuration:** The `mirror_config.env` allows you to define different "cleanup" rules for every repository.
* **Sanitation:** The Action automatically removes internal files and creates a "clean" commit before pushing.
* **Workflow:** You maintain your internal development in Gitea and use the `public` branch as