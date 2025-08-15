### **Chapter 5: Jenkins Jobs (Freestyle Projects)**
*The foundational build job type in Jenkins. Less flexible than Pipelines but ideal for simple tasks.*

#### **1. Creating Your First Freestyle Job**
*   **What it is:** A manually configured job type for building, testing, and deploying code without code-defined pipelines.
*   **Step-by-Step Creation:**
    1.  **Dashboard > New Item:** Enter job name (alphanumeric + underscores/dashes only; *avoid spaces!*), select "Freestyle project", click "OK".
    2.  **Description (Optional but Recommended):** Add purpose, owner, links to docs. *Critical for team collaboration.*
    3.  **General Settings:**
        *   *Restrict where this project can be run:* Specify label (e.g., `linux`, `docker-agent`) if using distributed builds.
        *   *Discard Old Builds:* **MANDATORY** to prevent disk exhaustion. Set `Max # of builds to keep` (e.g., 50) and/or `Days to keep builds` (e.g., 30).
        *   *GitHub project:* URL (e.g., `https://github.com/yourorg/repo/`) enables GitHub integration (status badges, links).
    4.  **Save:** Creates the job skeleton. *No SCM/build steps configured yet.*

#### **2. Configuring Source Code Management (SCM) – Git, SVN**
*   **Purpose:** Fetch source code before each build.
*   **Git Configuration (Most Common):**
    *   *Repository URL:* `https://github.com/yourorg/repo.git` or `git@github.com:yourorg/repo.git` (SSH). **⚠️ Avoid hardcoding credentials here!**
    *   *Credentials:* **Select a stored credential** (see Chapter 7) from the dropdown. *Never put username/password in the URL!*
    *   *Branches to build:* Default `*/main` or `*/master`. Use `*/feature-*` for feature branches. `**` builds *all* branches (rarely needed).
    *   *Additional Behaviors:*
        *   `Check out to specific local branch`: Needed for pushing back to Git (e.g., for version bumps).
        *   `Advanced clone behaviours`: Shallow clones (`--depth=1`), timeout settings.
        *   `Prune stale remote-tracking branches`: Keeps workspace clean (`git remote prune`).
*   **SVN Configuration:**
    *   *Repository URL:* `https://svn.example.com/repo/trunk` (or specific branch/tag).
    *   *Credentials:* Again, **use stored credentials**.
    *   *Check-out Depth:* `Fully recursive` (default), `Only top-level directory`, `User-specified depth`.
    *   *Excluded Regions:* Regex patterns to skip (e.g., `.*tests.*`).
*   **Critical Nuances:**
    *   **Workspace Location:** Code is checked out into `$JENKINS_HOME/workspace/<JOB_NAME>`. *Never assume absolute paths!*
    *   **Polling Impact:** SCM polling (see below) requires Jenkins to have read access to the repo.
    *   **SSH Key Setup:** For Git over SSH, the *Jenkins master's* SSH key (from `$JENKINS_HOME/.ssh/id_rsa`) must be added to GitHub/GitLab as a *deploy key* or the user's key must be stored in Jenkins Credentials.

#### **3. Build Triggers: Poll SCM, Build Periodically, Trigger Remotely**
*   **a) Poll SCM:**
    *   **What:** Jenkins periodically checks the SCM repository for *changes* (using `git fetch`/`svn info`).
    *   **Config:** `H/5 * * * *` (Jenkins syntax). Checks every 5 minutes. **⚠️ Heavy load on SCM server if many jobs!**
    *   **How it Works:** Compares latest commit in repo vs. last build's commit SHA. Triggers build *only if new commits exist*.
    *   **Best Practice:** Use sparingly. Prefer "GitHub Hook" triggers (requires GitHub Plugin) for instant notifications.
*   **b) Build Periodically (cron):**
    *   **What:** Schedule builds *regardless of code changes* (e.g., nightly builds, cleanup jobs).
    *   **Config:** Standard cron syntax (`MIN HOUR DOM MON DOW`). Examples:
        *   `0 2 * * *` : Every day at 2 AM.
        *   `H H/4 * * *` : Every 4 hours (staggered start times `H` prevents thundering herd).
        *   `0 0 1 * *` : First day of every month at midnight.
    *   **Key Insight:** *Triggers a build even if no code changes exist.* Useful for dependency updates, security scans, or reporting.
*   **c) Trigger Remotely (e.g., via URL):**
    *   **What:** Start a build via HTTP request (e.g., from scripts, other systems).
    *   **Config:** Enable checkbox, set `Authentication Token` (e.g., `my-secret-token`). *MUST set token for security!*
    *   **Trigger URL:** `JENKINS_URL/job/JOB_NAME/build?token=TOKEN_VALUE` (or `buildWithParameters` for parameters).
    *   **Security:** **⚠️ Exposing this URL without the token is a massive security risk!** Always use HTTPS. Restrict via firewall if possible.
    *   **Use Case:** Integration with chatops (Slack commands), external monitoring systems.

#### **4. Build Environment Settings**
*   **Purpose:** Configure the runtime environment *before* build steps execute.
*   **Key Options:**
    *   `Delete workspace before build starts`: **Use with extreme caution!** Wipes entire workspace (`rm -rf *`). Only for jobs needing pristine environments (e.g., complex builds with stale artifacts). *Slows down builds significantly.*
    *   `Use secret text(s) or file(s)`: Bind credentials as environment variables (see Chapter 7). *Safer than shell scripts accessing credentials directly.*
    *   `Add timestamps to the Console Output`: Critical for debugging timing issues. Format customizable.
    *   `Inject passwords to the build as environment variables`: Legacy method; prefer "Secret text/files" above.
    *   `Mask Passwords`: Hides specified strings (e.g., `API_KEY=123`) in console logs. *Not foolproof; avoid logging secrets!*
    *   `Provide Node & npm bin/ folder to PATH`: For Node.js jobs (requires NodeJS Plugin).
    *   `Abort the build if it's stuck`: Set inactivity timeout (e.g., `15 minutes`). Prevents hung builds.

#### **5. Executing Shell/Windows Batch Commands**
*   **The Core Build Step:** Where your actual build/test/deploy commands run.
*   **Shell (Unix/Linux Agents):**
    *   *Command:* Paste multi-line shell script. **Use `set -e` to fail fast on errors!**
    *   *Example:*
        ```bash
        #!/bin/bash
        set -e  # Critical: Fail immediately on error
        echo "Building version $BUILD_NUMBER"
        mvn clean package -DskipTests
        echo "Artifacts: $(ls target/*.jar)"
        ```
    *   **Key Env Vars:** `$WORKSPACE` (root dir), `$BUILD_NUMBER`, `$JOB_NAME`, `$GIT_COMMIT` (if SCM used).
*   **Windows Batch:**
    *   *Command:* `.bat` or `.cmd` commands. **Use `exit /B 1` to signal failure!**
    *   *Example:*
        ```batch
        @echo off
        setlocal enabledelayedexpansion
        echo Building version %BUILD_NUMBER%
        mvn clean package -DskipTests
        if errorlevel 1 exit /B 1
        echo Artifacts: %cd%\target\*.jar
        ```
*   **Critical Best Practices:**
    *   **Always fail fast:** `set -e` (Bash) or `if errorlevel 1 exit /B 1` (Batch).
    *   **Avoid absolute paths:** Use `$WORKSPACE`, `%WORKSPACE%`.
    *   **Log verbosely:** `echo` key steps for traceability.
    *   **Minimize commands:** Chain commands logically; avoid monolithic scripts.

#### **6. Post-Build Actions**
*   **Purpose:** Actions executed *after* the build steps complete (success, unstable, or failure).
*   **a) Archive Artifacts:**
    *   *What:* Save build outputs (JARs, logs, reports) for later download.
    *   *Config:* `Files to archive` (Ant-style pattern, e.g., `target/*.jar, logs/*.log`). `Archive even if the build is unstable`.
    *   **Why Essential:** Enables artifact deployment, debugging old builds, compliance.
    *   **⚠️ Pitfall:** Archiving massive files (e.g., entire workspace) fills `$JENKINS_HOME` disk. Be specific!
*   **b) Email Notifications:**
    *   *Basic Config:* `Recipients` (comma-separated emails), `Send separate emails...` (per failure), `Triggers` (Always, Success, Failure, Unstable).
    *   **Email Extension Plugin (Essential):** Replaces basic email. Offers:
        *   Customizable templates (HTML/Text).
        *   Recipient providers (developers, culprits, requestor).
        *   Attachments (artifacts, console logs).
        *   Advanced triggers (First Failure, Still Failing).
    *   **Critical Setup:** Requires global SMTP config (`Manage Jenkins > System > Extended E-mail Notification`).
*   **c) Trigger Downstream Jobs:**
    *   *What:* Start other jobs *after* this one finishes.
    *   *Config:* `Projects to build` (comma-separated job names), `Trigger only if build is...` (Stable, Unstable, Failure).
    *   *Pass Parameters:* `Block until the triggered projects finish their builds` (synchronous), `Pass parameters to downstream projects`.
    *   **Use Case:** Build → Test → Deploy pipeline stages.
    *   **⚠️ Pitfall:** Circular dependencies (Job A triggers Job B triggers Job A) cause infinite loops!

#### **7. Running and Monitoring Builds**
*   **Manual Trigger:** Click `Build Now` on job page.
*   **Build Queue:** View pending builds (`Build Queue` link on sidebar). Shows why builds are blocked (e.g., no executor).
*   **Build Progress:** Real-time console output (`Console Output` link during build). **Refresh manually!** (No auto-refresh by default).

#### **8. Viewing Build History and Console Output**
*   **Build History:** Left sidebar of job page. Shows:
    *   Build number, status (color-coded: Blue=Success, Yellow=Unstable, Red=Failure), duration, trigger reason.
    *   **Click a build number** for details.
*   **Console Output:**
    *   *Raw Text:* Full log (`Console Output` link). Searchable.
    *   *AnsiColor Plugin:* Renders ANSI colors (e.g., from `mvn` or `pytest`).
    *   **Debugging Tips:**
        *   Look for last successful command before failure.
        *   Check environment variables (`printenv` / `set`).
        *   Verify workspace paths (`pwd` / `cd`).
        *   Search for `ERROR`, `Exception`, `failed`.

---

### **Chapter 6: Jenkins Plugins**
*The ecosystem that extends Jenkins' core functionality.*

#### **1. Understanding Jenkins Plugins**
*   **What:** Modular components adding features (SCM, build tools, UI, integrations).
*   **Architecture:** Plugins are Java JARs. Core Jenkins provides extension points; plugins implement them.
*   **Dependency Hell:** Plugins depend on specific Jenkins core versions and *other plugins*. **⚠️ Major cause of upgrade failures!**

#### **2. Plugin Management**
*   **Installation:**
    1.  `Manage Jenkins > Plugins > Available Plugins`
    2.  Search, check box, `Install without restart` (preferred) or `Download now and install after restart`.
    3.  **Always review plugin description, version, compatibility, and security advisories!**
*   **Updating:**
    *   `Updates` tab shows available updates. **CRITICAL for security!**
    *   **Best Practice:** Update *one* plugin at a time. Test job functionality after *each* update. Check plugin changelogs.
*   **Removing:**
    1.  `Installed Plugins` tab > Find plugin > `Uninstall`.
    2.  **⚠️ NEVER delete plugin JARs manually from `$JENKINS_HOME/plugins`!** Causes corruption.
    3.  **Check Dependencies:** Uninstalling Plugin A might break Plugin B that depends on it. Jenkins warns you.

#### **3. Essential Plugins (Deep Dive)**
*   **a) Git Plugin:**
    *   **Why Essential:** Native Git integration (SCM, checkout, tagging, pushing).
    *   **Key Features:** Advanced clone behaviors, submodule support, Git executable management, lightweight checkouts.
    *   **Config:** Global Git settings (`Manage Jenkins > Tools > Git`), per-job SCM config (see Ch5).
*   **b) Email Extension Plugin:**
    *   **Why Essential:** Replaces weak built-in email. **Mandatory for professional notifications.**
    *   **Key Features:**
        *   Customizable templates (Groovy, Jelly, HTML).
        *   Recipient providers (`developers()`, `culprits()`, `requestor()`).
        *   Conditional content (`${FAILED_TESTS, ...}`).
        *   Attachments (`${FILE,path="target/report.html"}`).
    *   **Setup:** Global SMTP config + per-job advanced triggers/content.
*   **c) SSH Plugin:**
    *   **Why Essential:** Securely connect to remote servers (deployments, agent management).
    *   **Key Features:**
        *   `SSH Pipeline Steps` (for Declarative/Scripted Pipelines).
        *   `SSH Agent` build wrapper (for Freestyle) - injects SSH keys into build environment.
        *   Manage SSH credentials (see Ch7).
    *   **Use Case:** `sshCommand: remote: 'prod-server', command: 'sudo systemctl restart app'`.
*   **d) Pipeline Plugin:**
    *   **Why Essential:** **Foundation for modern Jenkins (code-defined pipelines).** *Not for Freestyle jobs!*
    *   **Components:**
        *   `Pipeline`: Core DSL (Declarative & Scripted).
        *   `Pipeline: Declarative Agent`: Agent directives.
        *   `Pipeline: Stage View`: Visual pipeline representation.
    *   **Critical Note:** Freestyle jobs are **legacy**. New projects should use Pipelines. This plugin enables them.
*   **e) Blue Ocean Plugin:**
    *   **Why Essential:** Modern UI for Pipelines (not Freestyle). **Not a replacement for classic UI!**
    *   **Key Features:**
        *   Visual pipeline editor (for Declarative Pipelines).
        *   Pipeline run visualization (stages, steps, logs).
        *   PR-focused workflow.
    *   **Limitation:** Poor Freestyle job support. Best for Pipeline-centric teams.
*   **f) Credentials Binding Plugin:**
    *   **Why Essential:** Securely inject credentials into builds *without exposing them in logs*. **Core security feature.**
    *   **How it Works:**
        *   Freestyle: `Build Environment > Use secret text(s) or file(s)`.
        *   Pipeline: `withCredentials([...]) { ... }`.
    *   **Mechanism:** Credentials are bound to environment variables *only* within the build step scope. Jenkins masks them in logs.
*   **g) Docker Plugin:**
    *   **Why Essential:** Run builds inside Docker containers (isolation, reproducibility).
    *   **Key Features:**
        *   `Docker Agent` - Run entire job in container.
        *   `Docker Build and Publish` build step - Build/push images.
        *   Connect to Docker hosts/registries via credentials.
    *   **Use Case:** `docker.build("myapp:$BUILD_ID").push()`.
*   **h) Kubernetes Plugin:**
    *   **Why Essential:** Dynamic agents on Kubernetes clusters (scalability, ephemeral environments).
    *   **How it Works:** Spins up a Pod per build (agent) using a template. Tears it down after.
    *   **Config:** Cloud settings (`Manage Jenkins > System > Kubernetes`), Pod templates (container specs).
    *   **Advantage:** No static agents needed; scales to zero.

#### **4. Best Practices for Plugin Usage**
*   **Minimalism:** Only install plugins you **actively need**. Each adds attack surface & complexity.
*   **Version Pinning:** In production, pin plugin versions in `plugins.txt` (for Docker/infra-as-code setups).
*   **Staging Environment:** **ALWAYS test plugin updates in a non-production Jenkins instance first.**
*   **Security Audits:** Regularly check `Manage Jenkins > Security > Plugin Vulnerability Warnings`.
*   **Avoid "Dead" Plugins:** Check update frequency & issue activity on plugin page. Stale plugins = security risks.
*   **Documentation:** Read the plugin's Wiki page *before* installing. Understand its purpose and config.

---

### **Chapter 7: Jenkins Credentials Management**
*The cornerstone of Jenkins security. **NEVER hardcode credentials!***

#### **1. Why Secure Credentials?**
*   **Risks of Hardcoding:**
    *   Accidental exposure in logs, source control, or job config XML.
    *   Impossible to rotate credentials without changing job configs everywhere.
    *   Violates security best practices (PCI-DSS, SOC2, ISO 27001).
*   **Jenkins Credentials Store:**
    *   Centralized, encrypted storage (AES-256 by default).
    *   Access controlled by Jenkins permissions.
    *   Credentials are **never** stored in plaintext in job configs.

#### **2. Types of Credentials**
| Type               | Use Case                                      | Storage Format                     | Security Note                                  |
| :----------------- | :-------------------------------------------- | :--------------------------------- | :--------------------------------------------- |
| **Username/Password** | Git, SVN, Docker Registry, Artifactory, etc. | Encrypted username + password     | **Most common.** Avoid for production secrets. |
| **SSH Username with private key** | Git (SSH), SSH Plugin, remote servers      | Encrypted private key (or passphrase) | **Preferred for Git.** Store key *in Jenkins*, not on disk. |
| **Secret Text**      | API Tokens (GitHub, Jira), Passphrases      | Encrypted text string             | **Use for tokens/secrets.** Safer than username/password combos. |
| **Certificate**      | TLS/SSL Client Certificates                 | Encrypted PKCS#12 file + password | Rare. Requires password for the cert file itself. |
| **Docker Server Certificate Authentication** | Secure Docker Daemon (TLS)                | Cert + Key + CA files             | For `docker-plugin` connecting to secure Docker hosts. |
| **Username with SSH key** | Alternative SSH credential type           | Username + key (like SSH type)    | Functionally similar to "SSH Username...".     |

*   **Critical Insight:** `Secret Text` is often the **most secure** option for API tokens (e.g., GitHub Personal Access Tokens), as it avoids storing a username context.

#### **3. Adding and Managing Credentials**
*   **Where:** `Manage Jenkins > Credentials > System > Global credentials (unrestricted)` (or Folder-specific).
*   **Adding a Credential:**
    1.  `Add Credentials` (on right sidebar).
    2.  **Kind:** Select type (e.g., `Username with password`).
    3.  **Scope:** `Global` (all jobs) or `System` (Jenkins-internal, e.g., for plugins). *Prefer `Global` for job usage.*
    4.  **Username:** (If applicable) e.g., `gitlab-deploy-user`, `jenkins@myorg.com`.
    5.  **Password/Secret/Key:** Paste the value. **⚠️ NEVER use your personal password! Use service accounts.**
    6.  **ID:** **CRITICAL!** Auto-generated, but **rename to meaningful, lowercase, hyphenated ID** (e.g., `github-deploy-token`, `artifactory-read`). *This is what you reference in jobs.*
    7.  **Description:** Clear purpose (e.g., "Read-only token for Maven deps").
*   **Managing Credentials:**
    *   **Rotation:** Update the secret value *in place* via Jenkins UI. Jobs using the ID **automatically get the new value** on next build. *No job config changes needed!*
    *   **Deletion:** Only delete credentials no longer referenced *anywhere*. Check `Used By` column.
    *   **Folders:** Use Folder-based credentials for team isolation (requires "CloudBees Folders Plugin").

#### **4. Using Credentials in Jobs and Pipelines**
*   **a) Freestyle Jobs:**
    *   **SCM (Git):** In repo config, select credential from dropdown (ID/Description shown).
    *   **Build Environment:** `Use secret text(s) or file(s)`:
        *   `Specific credentials` > Select credential > Bind to environment variable (e.g., `MY_API_TOKEN`).
        *   **Result:** `echo $MY_API_TOKEN` in shell step (masked in logs!).
    *   **SSH Plugin:** `SSH Agent` build wrapper > Select SSH key credential.
*   **b) Pipelines (Declarative Example):**
    ```groovy
    pipeline {
        agent any
        environment {
            // Bind Secret Text credential to ENV var
            API_KEY = credentials('github-deploy-token')
        }
        stages {
            stage('Build') {
                steps {
                    sh 'echo Using token: $API_KEY' // Token masked in logs!
                    // OR use withCredentials for finer control:
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'artifactory-creds',
                            usernameVariable: 'ARTIFACTORY_USER',
                            passwordVariable: 'ARTIFACTORY_PASS'
                        )
                    ]) {
                        sh 'curl -u $ARTIFACTORY_USER:$ARTIFACTORY_PASS https://artifactory.example.com'
                    }
                }
            }
        }
    }
    ```
*   **c) Docker Plugin:**
    *   In `Docker Build and Publish` step: Select credential ID for registry (`dockerhub-creds`).
*   **d) Kubernetes Plugin:**
    *   In Pod Template: Set `Image Pull Secret` to credential ID for private registry.

#### **5. Securing Credentials with Jenkins Credentials Store**
*   **Encryption:** Credentials are encrypted at rest using a key stored in `$JENKINS_HOME/secret.key`. **Back up this key!** (Required for restoring Jenkins).
*   **Access Control:**
    *   **Mandatory:** Enable Security (`Manage Jenkins > Security`). Use `Matrix Authorization Strategy` or `Role-Based Strategy`.
    *   **Principle of Least Privilege:** Restrict `Credentials > View`, `Credentials > Create`, `Credentials > Update`, `Credentials > Delete` to only admin/trusted roles.
    *   **Folders:** Isolate credentials by team/project using Folders.
*   **Critical Security Practices:**
    *   **Service Accounts Only:** Credentials should belong to service accounts (e.g., `jenkins-build`), **NEVER personal accounts**.
    *   **Least Privilege:** Credentials should have *only* the permissions needed (e.g., Git read-only, Artifactory deploy role).
    *   **Rotation Policy:** Rotate credentials regularly (e.g., tokens every 90 days). Jenkins makes this painless.
    *   **Audit Logs:** Enable audit logging (`audit2-plugin`) to track credential access/modification.
    *   **Avoid "System" Scope:** `System` credentials are for Jenkins internals (e.g., GitHub OAuth). Don't store job secrets here.
    *   **Secret Text > Passwords:** Prefer `Secret Text` for tokens over `Username/Password` where possible.

---
