# Exercise 8: Creating a "Hello World" Jenkins Job

## Objective

Store a simple shell script in a GitHub repository, then create a Jenkins Freestyle job that pulls the repository and runs the script.

## Prerequisites

- Jenkins running in a Docker container and reachable at `http://localhost:8080` (see the Jenkins installation exercise)
- A GitHub account
- Git installed locally

---

## Part 1: Put the script on GitHub

### 1. Create a GitHub repository

1. Log in to [github.com](https://github.com).
2. Click **+** (top right) → **New repository**.
3. Fill in:
   - **Repository name:** `devops-sample-code`
   - **Description:** A demo repository for Jenkins scripting.
   - **Visibility:** Public
4. Click **Create repository**.

Repository used: `https://github.com/NehaBhask/devops-sample-code.git`

### 2. Create a fine-grained personal access token (PAT)


### 3. Create the script locally

```bash
touch hello-world.sh
```

Add this content to the file:

```bash
#!/bin/bash
echo "Hello, Jenkins!"
```

Make it executable:

```bash
chmod +x hello-world.sh
```

### 4. Initialize a local Git repository

```bash
git init
```

### 5. Add and commit the script

```bash
git add --chmod=+x -- hello-world.sh
git status
git commit -m "Add hello-world.sh"
```

`git status` should show `hello-world.sh` as a new file ready to be committed.

### 6. Link the local repo to GitHub and push

```bash
git remote add origin https://github.com/NehaBhask/devops-sample-code.git
git push -u origin master
```

When prompted, enter your GitHub username, and use the PAT as the password.

> The branch name must match your local branch. If your branch is `main`, push `main` instead (check with `git branch`).

### 7. Verify on GitHub

Open `https://github.com/NehaBhask/devops-sample-code` and confirm `hello-world.sh` is listed with the commit message **Add hello-world.sh**.

---

## Part 2: Create the Jenkins job
As done in exercise 7

### Step 1: Access Jenkins

Open `http://localhost:8080` and log in with your admin credentials.

### Step 2: Create a new job

1. On the dashboard, click **New Item**.
2. Enter the name `HelloWorld`.
3. Select **Freestyle project**.
4. Click **OK**.

### Step 3: Configure the job

**General**
- Description: `Hello World! Jenkins job.`

**Source Code Management**
- Select **Git**.
- Repository URL: `https://github.com/NehaBhask/devops-sample-code.git`
- Credentials: none (the repository is public)
- Branch Specifier: `*/master` (use `*/main` if your repository's branch is `main`)

**Build Steps**
- Click **Add build step** → **Execute shell**.
- Enter:

```bash
sh hello-world.sh
```

### Step 4: Save and run

1. Click **Save**.
2. On the job page, click **Build Now**.

### Step 5: View the build output

1. In **Build History**, click the build number (`#1`).
2. Click **Console Output**.

---

## Result: Console output

```
Started by user admin
Running as SYSTEM
Building in workspace /var/jenkins_home/workspace/HelloWorld
The recommended git tool is: NONE
No credentials specified
Cloning the remote Git repository
Cloning repository https://github.com/NehaBhask/devops-sample-code.git
 > git init /var/jenkins_home/workspace/HelloWorld # timeout=10
Fetching upstream changes from https://github.com/NehaBhask/devops-sample-code.git
 > git --version # timeout=10
 > git --version # 'git version 2.47.3'
 > git fetch --tags --force --progress -- https://github.com/NehaBhask/devops-sample-code.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git config remote.origin.url https://github.com/NehaBhask/devops-sample-code.git # timeout=10
 > git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/* # timeout=10
Avoid second fetch
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
Checking out Revision e47066e1e228a4d6db449c65ac167cf18cf67d8f (refs/remotes/origin/master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f e47066e1e228a4d6db449c65ac167cf18cf67d8f # timeout=10
Commit message: "Add hello-world.sh"
First time build. Skipping changelog.
[HelloWorld] $ /bin/sh -xe /tmp/jenkins4275070433741655195.sh
+ sh hello-world.sh
Hello, Jenkins!
Finished: SUCCESS
```

## What the log shows

| Log section | Meaning |
|-------------|---------|
| `Started by user admin` | The build was triggered manually via **Build Now** |
| `Building in workspace /var/jenkins_home/workspace/HelloWorld` | Jenkins created a workspace for the job inside the container |
| `No credentials specified` | No login was needed because the repo is public |
| `Cloning repository ...` / `git fetch ...` | Jenkins pulled the code from GitHub |
| `Checking out Revision e47066e... (refs/remotes/origin/master)` | It checked out the latest commit on `master` |
| `Commit message: "Add hello-world.sh"` | This is the commit created in Part 1 |
| `First time build. Skipping changelog.` | There is no previous build to compare against |
| `+ sh hello-world.sh` | The Execute shell step ran the script (`-x` echoes each command) |
| `Hello, Jenkins!` | Output of the script |
| `Finished: SUCCESS` | The build passed |


