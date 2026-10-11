# Exercise 9: Jenkins Multi-Stage Pipeline Exercise: Deploying a Python Flask Application

**Repository:** https://github.com/NehaBhask/devops-sample-code
**Jenkins job:** `Python-MultiStage-Pipeline`
**Environment:** Jenkins LTS running in Docker (`jenkins/jenkins:lts`, Debian 13 "trixie" base), host on Windows (PowerShell)

---

## 1. Objective

Create and run a Jenkins declarative pipeline that automates CI/CD for a sample Python Flask application with these stages:

| Stage | Purpose |
|-------|---------|
| Build | Prepare the environment / install dependencies |
| Test | Run unit tests |
| Deploy | Copy the application to a mock deployment directory |
| Run Application | Start the app in the background |
| Test Application | Re-run tests against the running setup |

---

## 2. Sample Application

Files in the `python-flask-app` project:

**`app.py`**
```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello, Jenkins Multi-Stage Pipeline!"

if __name__ == "__main__":
    app.run(debug=True)
```

**`requirements.txt`**
```
flask==2.1.2
```

**`test_app.py`**
```python
import unittest
from app import app

class TestApp(unittest.TestCase):
    def test_home(self):
        tester = app.test_client()
        response = tester.get("/")
        print(response.data.decode("utf-8"))
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.data.decode("utf-8"), "Hello, Jenkins Multi-Stage Pipeline!")

if __name__ == "__main__":
    unittest.main()
```

The code was pushed to GitHub (`git init`, `git add .`, `git commit`, `git remote add origin ...`, `git push`).

---

## 3. Jenkins Job Setup

1. Jenkins dashboard → **New Item** → name `Python-MultiStage-Pipeline` → **Pipeline** → OK.
2. Under **Pipeline**, selected **Pipeline script from SCM**.
3. SCM: **Git**, repository URL `https://github.com/NehaBhask/devops-sample-code`, no credentials (public repo).
4. Branch: `master`. Script path: `Jenkinsfile`.
5. Saved and clicked **Build Now**.

### Initial Jenkinsfile

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Creating virtual environment and installing dependencies...'
            }
        }
        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'python3 -m unittest discover -s .'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying application...'
                sh '''
                mkdir -p ${WORKSPACE}/python-app-deploy
                cp ${WORKSPACE}/app.py ${WORKSPACE}/python-app-deploy/
                '''
            }
        }
        stage('Run Application') {
            steps {
                echo 'Running application...'
                sh '''
                nohup python3 ${WORKSPACE}/python-app-deploy/app.py > ${WORKSPACE}/python-app-deploy/app.log 2>&1 &
                echo $! > ${WORKSPACE}/python-app-deploy/app.pid
                '''
            }
        }
        stage('Test Application') {
            steps {
                echo 'Testing application...'
                sh '''
                python3 ${WORKSPACE}/test_app.py
                '''
            }
        }
    }

    post {
        success { echo 'Pipeline completed successfully!' }
        failure { echo 'Pipeline failed. Check the logs for more details.' }
    }
}
```

---

## 4. Result of the First Run: FAILURE

**Commit built:** `a4a877e` ("Add Jenkinsfile") – first build, so the repo was cloned fresh.

The **Build** stage passed (it only echoes a message), but the **Test** stage failed:

```
+ python3 -m unittest discover -s .
.../script.sh.copy: 1: python3: not found
ERROR: script returned exit code 127
Finished: FAILURE
```

The Deploy, Run Application and Test Application stages were skipped due to the earlier failure, and the `post { failure { ... } }` block printed:
`Pipeline failed. Check the logs for more details.`

### Root cause

Exit code **127** means "command not found". The official `jenkins/jenkins:lts` image does not ship with Python, so `python3` (and `pip`, Flask) were not available to the pipeline.

---

## 5. Fix: Installing Python Inside the Jenkins Container

### 5.1 Find the container and open a root shell

```powershell
docker ps -a
docker exec -it -u root 560e5451e7b4 bash
```

(`-u root` is needed because the default `jenkins` user cannot install packages.)

### 5.2 Install the dependencies

Run inside the container (`root@560e5451e7b4:/#`):

```bash
apt-get update
apt install python3
apt install pip            # apt resolved this to python3-pip
apt install python3.11-venv
apt install python3-flask
```

### 5.3 What happened at each step

| Command | Outcome |
|---------|---------|
| `apt-get update` | Succeeded. Fetched 10.2 MB of package lists from the Debian **trixie** repositories. |
| `apt install python3` | Succeeded. Installed **Python 3.13.5** plus 10 dependencies (11 packages, ~23.4 MB). |
| `apt install pip` | Succeeded. apt selected `python3-pip` (25.1.1) and pulled in 72 packages including `build-essential`, `gcc`, `g++`, `make` and `python3-dev` (~364 MB). |
| `apt install python3.11-venv` | **Failed**: `Unable to locate package python3.11-venv`. |
| `apt install python3-flask` | Installed Flask and 16 dependencies (Werkzeug, Jinja2, Click, etc., ~39 MB). |

### 5.4 Note on the `python3.11-venv` error

This error is expected here, not a real problem. The container is based on Debian 13 (trixie), which ships **Python 3.13**, so no `python3.11-*` packages exist. If a virtual environment is ever needed, the correct package is the version-agnostic:

```bash
apt install python3-venv
```

It was not required for this exercise because Flask was installed system-wide through apt (`python3-flask`) rather than through `pip install -r requirements.txt` in a venv.

---

## 6. Result of the Final Run: SUCCESS

**Commit built:** `ca87126` ("change in jenkinsfile"), branch `master`, git 2.47.3.

### Changes to the repo/Jenkinsfile before the successful run

Compared with the original exercise, the logs show these adjustments:

| Item | Original | Final (from logs) |
|------|----------|-------------------|
| Test discovery | `python3 -m unittest discover -s .` | `python3 -m unittest discover -s tests -p test*.py -v` |
| Test file location | repo root | `tests/` directory |
| App location | `app.py` in repo root | `python-flask-app/app.py` |
| Test Application stage | `python3 test_app.py` | `python3 -m unittest discover -s tests -p test*.py -v` |

### Stage-by-stage output

| Stage | Result | Evidence |
|-------|--------|----------|
| Declarative: Checkout SCM | Passed | Checked out `ca87126` from `origin/master` |
| Build | Passed | Echo: `Creating virtual environment and installing dependencies...` |
| Test | Passed | `test_home (test_app.TestApp.test_home) ... ok`, `Ran 1 test in 0.007s`, `OK` |
| Deploy | Passed | Created `python-app-deploy/` and copied `python-flask-app/app.py` into it |
| Run Application | Passed | Launched `nohup python3 .../python-app-deploy/app.py` in the background (PID `2676`) |
| Test Application | Passed | Same test passed again: `Ran 1 test in 0.008s`, `OK` |
| Post Actions | Passed | `Pipeline completed successfully!` |

### Key console excerpts

```
[Pipeline] { (Test)
+ python3 -m unittest discover -s tests -p test*.py -v
test_home (test_app.TestApp.test_home) ... ok

----------------------------------------------------------------------
Ran 1 test in 0.007s

OK
Hello, Jenkins Multi-Stage Pipeline!
```

```
[Pipeline] { (Deploy)
+ mkdir -p /var/jenkins_home/workspace/Python-MultiStage-Pipeline/python-app-deploy
+ cp /var/jenkins_home/workspace/Python-MultiStage-Pipeline/python-flask-app/app.py /var/jenkins_home/workspace/Python-MultiStage-Pipeline/python-app-deploy/
```

```
Pipeline completed successfully!
Finished: SUCCESS
```

---

## 7. Failed vs. Successful Run: Comparison

| | Run 1 | Run 2 |
|---|---|---|
| Commit | `a4a877e` "Add Jenkinsfile" | `ca87126` "change in jenkinsfile" |
| Python in container | Not installed | Python 3.13 + Flask installed |
| Test stage | `python3: not found` (exit 127) | 1 test ran, OK |
| Later stages | Skipped | All executed |
| Final status | **FAILURE** | **SUCCESS** |

---


