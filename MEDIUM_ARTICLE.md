# Building a Git and GitHub Workflow from Scratch with WSL 2

## A hands-on DevOps foundation using Ubuntu, Git, SSH authentication, and GitHub

Reading about Git commands is useful, but the concepts became much clearer when I built the entire workflow and watched a file move through each state myself.

In this project, I started with a Windows workstation, created a Linux development environment with WSL 2 and Ubuntu 24.04 LTS, configured Git, created an empty GitHub repository, recorded the first commit, established SSH authentication, and published the `main` branch.

The final repository is intentionally simple. It contains one required project file plus the documentation. The value of the project is not the complexity of the file. The value is understanding the system around it: the working tree, staging area, local repository, remote repository, commit identity, SSH authentication, and validation steps.

This article explains what I built, why I made each decision, the problems I encountered, and what the project taught me.

## 1. Project Architecture and Objectives

The development path was:

```text
Rester McGlown Jr.
        |
        v
Windows Terminal
        |
        v
WSL 2 -> Ubuntu 24.04 LTS -> Bash -> Git
                                      |
                                      | SSH
                                      v
                         GitHub: rester2007/Git-Project-1
```

Inside Git, the file followed a second path:

```text
Working tree       Staging area       Local repository       GitHub
   file.txt  --git add-->  index  --git commit-->  main  --git push--> origin/main
```

My objectives were to:

- work in a Linux environment while retaining my Windows workstation;
- install and configure Git with an intentional identity and branch standard;
- create and clone a GitHub repository;
- create the required file from the command line;
- observe the working tree and staging area before committing;
- create a clear root commit;
- authenticate to GitHub with an SSH key instead of a password;
- push `main` and verify that the remote branch matched the local commit; and
- document the project so another learner could reproduce it.

The project is a learning environment, not a production application. That distinction matters. Good portfolio documentation should accurately represent what was built instead of making a foundational exercise sound like a large platform.

## 2. Phase 1: Building a Linux Development Environment with WSL 2

Most cloud workloads run on Linux, so I wanted the project commands and filesystem behavior to come from a Linux environment. I used WSL 2 because it gives Windows users a practical Linux kernel and Ubuntu user space without requiring a separate dual-boot installation or a manually managed desktop virtual machine.

From PowerShell, I updated and inspected WSL:

```powershell
wsl --update
wsl --status
wsl --list --verbose
```

The project used a pinned Ubuntu 24.04 LTS distribution named `Ubuntu-24.04-Dev`. Pinning the distribution made the environment explicit instead of depending on whichever Ubuntu release happened to be the current default.

Inside Ubuntu, I confirmed the active account and working directory:

```bash
whoami
pwd
sudo -v
```

The results were:

```text
rester
/home/rester
```

These basic checks serve different purposes. `whoami` confirms the effective user. `pwd` confirms where commands will operate. `sudo -v` validates administrative credentials without making a system change.

I then updated the package metadata and installed updates before installing Git:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git
git --version
```

The completed environment reported Git 2.43.0.

I kept the repository under `/home/rester` instead of placing it under `/mnt/c`. This preserves Linux-native paths, permissions, and line-ending behavior and avoids crossing between the Linux and Windows filesystems for every Git operation.

## 3. Phase 2: Configuring Git Identity and Repository Defaults

Git records an author name and email in every commit. I configured them globally because this Ubuntu distribution is my development environment:

```bash
git config --global user.name "rester"
git config --global user.email "rmcglown@hotmail.com"
git config --global init.defaultBranch main
git config --global core.autocrlf input
```

I reviewed the results with:

```bash
git config --global --list
git config --list --show-origin
```

Each setting has a specific role:

- `user.name` and `user.email` become commit metadata.
- `init.defaultBranch main` gives new repositories a consistent initial branch name.
- `core.autocrlf input` converts Windows-style CRLF endings to LF when content is committed while keeping Linux LF endings in the working tree.

One important distinction is that the Git email is not a login. It identifies the author of a commit; it does not authenticate a push to GitHub. Authentication was configured separately with SSH.

Because this is a public repository, the author email can be visible in Git history. I chose to use my email for this project, but GitHub's no-reply email is an alternative when privacy is preferred.

## 4. Phase 3: Creating the Repository and Understanding Git States

I created a new empty public repository named `Git-Project-1` under the GitHub account `rester2007`. I left the initialization options unchecked so the first commit would be created from the command line.

I cloned the empty repository and entered it:

```bash
cd ~
git clone https://github.com/rester2007/Git-Project-1.git
cd Git-Project-1
pwd
```

Cloning did more than create a folder. It also created the `.git` metadata directory, registered GitHub as the remote named `origin`, and established the repository relationship.

Next, I created the required file:

```bash
printf '%s\n' 'new file for repository' > file.txt
cat file.txt
```

I used `printf` because its output is predictable and I could explicitly include the final newline. The `>` operator redirected the output into `file.txt`.

Then I inspected the repository:

```bash
git status
```

Git reported `file.txt` as untracked. The file existed in the filesystem, but it was not yet part of Git's planned snapshot.

I staged it:

```bash
git add file.txt
git status
git diff --cached
```

This was the most important conceptual point in the exercise. `git add` did not publish anything, and it did not create permanent history. It copied the selected version of the file into the staging area, also called the index.

`git diff --cached` showed the exact content that the next commit would record:

```diff
diff --git a/file.txt b/file.txt
new file mode 100644
--- /dev/null
+++ b/file.txt
@@ -0,0 +1 @@
+new file for repository
```

Only after reviewing the staged snapshot did I create the commit:

```bash
git commit -m "Add initial repository file"
```

Git created the root commit:

```text
ca38536 Add initial repository file
```

A root commit is the first commit in a repository. Unlike later commits, it has no parent commit.

I validated it with:

```bash
git status
git log --oneline --decorate --stat -1
```

At this point the working tree was clean. That meant the tracked files matched the checked-out commit. It did not yet prove that GitHub contained the commit.

## 5. Phase 4: Securing GitHub Access with SSH

For command-line authentication, I generated an Ed25519 SSH key pair:

```bash
ssh-keygen -t ed25519 -C "rmcglown@hotmail.com"
```

The default key pair consisted of:

```text
~/.ssh/id_ed25519       private key
~/.ssh/id_ed25519.pub   public key
```

These two files must be treated differently. The public key is registered with GitHub. The private key stays on the workstation and should never be pasted into a website, sent to another person, or committed to a repository.

After adding the public key under GitHub's SSH and GPG key settings, I tested the connection:

```bash
ssh -T git@github.com
```

On the first connection, SSH displayed GitHub's host-key fingerprint. This check protects the server side of the connection. After validating and accepting the fingerprint, SSH saved the host identity in `~/.ssh/known_hosts`.

The successful response identified the account:

```text
Hi rester2007! You've successfully authenticated, but GitHub does not provide shell access.
```

The final sentence is not an error. GitHub accepts SSH for Git operations but does not provide a general interactive Linux shell.

I then changed `origin` from HTTPS to SSH:

```bash
git remote set-url origin git@github.com:rester2007/Git-Project-1.git
git remote -v
```

Finally, I published the branch:

```bash
git push -u origin main
```

The `-u` option connected the local `main` branch to the remote `origin/main` branch. That upstream relationship allows later commands to use `git push` or `git pull` without repeating the remote and branch names.

## 6. Validation: Proving the Local and Remote States Match

I did not treat the push message alone as the final proof. I checked the repository from several angles:

```bash
git status
git branch -vv
git remote -v
git log --oneline --decorate --stat -1
git rev-parse HEAD
git ls-remote origin refs/heads/main
```

The important results were:

| Validation | Result |
|---|---|
| Local branch | `main` |
| Upstream | `origin/main` |
| Root commit | `ca385369ea8f8ed3629f77071a9778df8e20758f` |
| Required file | `file.txt` |
| Required content | `new file for repository` |
| GitHub account | `rester2007` |
| SSH authentication | Successful |
| Working tree | Clean after the first push |

The full hash from `git rev-parse HEAD` matched the hash returned for GitHub's `main` reference. This demonstrated that the local branch and remote branch pointed to the same commit.

## 7. Troubleshooting: Three Problems and What They Taught Me

### Problem 1: An unmatched quote changed the Bash prompt

My first SSH key command was missing its closing quotation mark:

```text
ssh-keygen -t ed25519 -C "rmcglown@hotmail.com
>
```

Bash displayed `>` and waited. In this context, `>` was the continuation prompt. The shell had not run `ssh-keygen`; it was waiting for the rest of the quoted string.

I cancelled the incomplete command with `Ctrl+C` and entered it again with matching quotes:

```bash
ssh-keygen -t ed25519 -C "rmcglown@hotmail.com"
```

The lesson was to interpret the prompt before typing more input. An unexpected continuation prompt usually means the command is syntactically incomplete.

### Problem 2: The upstream branch was reported as gone

After the first local commit, `git status` reported that the branch was based on `origin/main`, but the upstream was gone.

The repository had been cloned while empty. There was no `main` reference on GitHub yet, so the local configuration referred to a remote branch that did not exist. The first push created it:

```bash
git push -u origin main
```

The lesson was that a local commit and a remote branch are separate objects. Committing does not automatically create or update the GitHub branch.

### Problem 3: Commit identity and remote authentication looked similar but were different

Git stored my name and email without asking GitHub for permission, but pushing required SSH authentication.

That behavior reinforced an important separation:

- commit identity answers, “Who does this commit claim as its author?”
- SSH authentication answers, “Is this connection authorized to write to this GitHub account?”

In a professional workflow, both must be configured correctly, but one does not replace the other.

## 8. Security Review

Even a small Git exercise involves credentials and public metadata. I applied the following practices:

- kept the private SSH key only in the Linux account;
- uploaded only the public `.pub` key to GitHub;
- kept the private key out of commands, documentation, and the repository;
- reviewed GitHub's host fingerprint during the first connection;
- inspected staged content before committing;
- kept secrets and credentials out of repository files and remote URLs; and
- treated the commit email as public metadata.

SSH was the best fit for this Linux-focused project because it reinforced a skill also used when administering servers. A passphrase can protect the private key at rest, and `ssh-agent` can be used to avoid repeatedly entering that passphrase during an active session. HTTPS with a credential manager would also have been a valid alternative. The important engineering decision is to use a supported credential mechanism without storing secrets in plaintext.

## 9. Key Lessons and Future Improvements

The project reinforced several lessons:

- Git is local first. GitHub is a remote hosting and collaboration platform.
- Saving, staging, committing, and pushing are separate operations.
- The staging area provides control over exactly what enters a commit.
- A clean working tree is not the same as a synchronized remote repository.
- Remote-tracking references represent the last known remote state; they are not live views.
- SSH authentication uses the private key locally and the public key registered remotely.
- Clear commit messages and validation commands make even a small project easier to audit.
- A Linux development environment on WSL 2 provides experience that transfers to cloud-hosted Linux systems.

The logical next steps are to create a feature branch, open and merge a pull request, resolve a controlled merge conflict, add an appropriate `.gitignore`, and introduce a GitHub Actions workflow when the repository contains testable code.

## 10. Conclusion

Git Project 1 established the complete version-control path I will reuse in larger DevOps and cloud projects:

```text
create -> inspect -> stage -> review -> commit -> authenticate -> push -> verify
```

The final file was simple, but the workflow was complete. I built the Linux environment, configured Git intentionally, learned how content moves through Git's states, separated authorship from authentication, secured GitHub access with SSH, and verified that the local and remote branches referenced the same commit.

That foundation matters because infrastructure code, automation scripts, configuration files, documentation, and CI/CD workflows all depend on disciplined version control.

## Project Repository

[GitHub: rester2007/Git-Project-1](https://github.com/rester2007/Git-Project-1)

## References

- [Microsoft: Install WSL](https://learn.microsoft.com/en-us/windows/wsl/install)
- [Microsoft: Set up a WSL development environment](https://learn.microsoft.com/en-us/windows/wsl/setup/environment)
- [Git Reference Manual](https://git-scm.com/docs)
- [GitHub: Creating and managing repositories](https://docs.github.com/en/repositories/creating-and-managing-repositories)
- [GitHub: Connecting to GitHub with SSH](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)

---

**Rester McGlown Jr.** is a DevOps and Cloud Engineer building hands-on projects in Linux, AWS, automation, infrastructure, and operational security.

- [LinkedIn](https://www.linkedin.com/in/rester-mcglown-jr)
- [Medium](https://medium.com/@rester.mcglown)
- [GitHub project](https://github.com/rester2007/Git-Project-1)
