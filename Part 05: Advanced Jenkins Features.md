### **Chapter 13: Jenkins Agents (Slaves)**  
*Decoupling build execution from the controller for scalability and isolation.*

#### **1. Master-Agent Architecture**  
- **Why it exists**:  
  Jenkins Controller (Master) handles UI, scheduling, and coordination. **Agents** (formerly "Slaves") execute builds. Prevents controller overload and enables OS/environment isolation.  
- **Key Components**:  
  - **Controller**: Manages jobs, agents, plugins, security. *Never runs builds by default*.  
  - **Agent**: Dedicated machine (physical/virtual/container) running `agent.jar`. Connects to controller via TCP.  
  - **Communication**: Encrypted TCP (JNLP port 50000 by default). Controller initiates connections *to* agents (reverse connection).  
- **Critical Insight**:  
  > ⚠️ **Controller != Build Machine**: If builds run on the controller (`Built-In Node`), it risks resource starvation. *Always use agents for production builds*.

#### **2. Setting Up Agent Nodes**  
##### **Permanent Agents**  
- **Use Case**: Stable workloads (e.g., Windows builds, GPU jobs).  
- **Setup Steps**:  
  1. *Controller*: `Manage Jenkins > Nodes > New Node` > Permanent Agent.  
  2. Configure:  
     - `Name`: `linux-builder-01`  
     - `Remote root directory`: `/home/jenkins/agent` (must exist on agent)  
     - `Labels`: `linux docker` (for job targeting)  
     - `Usage`: "Utilize this node as much as possible" (default)  
  3. **Launch Method**:  
     - **SSH**: Controller SSHs into agent (requires credentials). *Best for on-prem*.  
     - **JNLP**: Agent runs `java -jar agent.jar -jnlpUrl http://controller:8080/computer/agent-name/slave-agent.jnlp -secret [token]`. *Best for firewalled environments*.  
     - **Manual**: Copy `agent.jar` from `http://controller:8080/jnlpJars/agent.jar` and run manually.  
- **Agent Service Setup**:  
  Use `Install as a service` option in agent config to auto-start on boot (creates systemd/upstart job).

##### **Ephemeral Agents**  
- **Use Case**: Short-lived builds (e.g., CI pipelines), cost optimization.  
- **Mechanism**:  
  - Spun up *on-demand* (e.g., Docker container, EC2 instance).  
  - Terminated after build/idle timeout.  
- **Setup**:  
  - Requires **cloud plugin** (e.g., Docker, EC2). *Configured in Chapter 14*.

#### **3. Configuring Agents via Protocols**  
| **Method** | **How It Works** | **When to Use** | **Security Notes** |  
|------------|------------------|-----------------|-------------------|  
| **SSH** | Controller SSHs into agent using credentials. Runs `agent.jar` remotely. | On-prem agents, static IPs. | Use SSH keys (not passwords). Restrict user permissions. |  
| **JNLP** | Agent initiates connection to controller (port 50000). | Firewalled agents, cloud VPCs. | Firewall must allow *outbound* from agent to controller. Use `-tunnel` for NAT. |  
| **Docker** | Controller starts container via Docker API. Agent runs inside container. | Dynamic scaling, ephemeral builds. | Secure Docker daemon (TLS). Use non-root user in container. |  

- **JNLP Deep Dive**:  
  - Agent command:  
    ```bash
    java -jar agent.jar -jnlpUrl http://jenkins:8080/computer/my-agent/slave-agent.jnlp \
          -secret abc123def456 -workDir /var/jenkins_agent
    ```  
  - `-workDir`: Overrides default agent workspace (avoids `/tmp` deletion).  
  - `-tunnel controller-ip:50000`: Bypasses NAT/firewall restrictions.

#### **4. Labeling and Distributing Workloads**  
- **Labels**: Tags assigned to agents (e.g., `linux`, `docker`, `gpu`).  
- **Job Targeting**:  
  - Freestyle Job: `Restrict where this project can be run` > Label Expression.  
  - Pipeline:  
    ```groovy
    pipeline {
      agent { label 'docker && linux' } // Uses agents with BOTH labels
      stages { ... }
    }
    ```  
- **Advanced Labeling**:  
  - Boolean logic: `linux || mac`, `docker && !arm64`  
  - **Exclusive Execution**: Use `Lockable Resources` plugin to reserve agents (e.g., for hardware-in-loop tests).

#### **5. Cloud-Based Agents**  
*Agents provisioned dynamically via cloud providers.*  
- **EC2 (AWS)**:  
  - Plugin: **Amazon EC2**  
  - Setup:  
    1. Configure AWS credentials in Jenkins (IAM role with `ec2:RunInstances`).  
    2. Define *Template*: AMI ID, instance type (`t3.medium`), security group (open port 50000).  
    3. Set `Idle Termination Minutes` (e.g., 15 mins after last build).  
  - **Spot Instances**: Enable `Use Spot Instance` to save 70-90% cost. *Handle interruptions with `Instance interruption behavior`*.  
- **GCE (Google Cloud)**:  
  - Plugin: **Google Compute Engine**  
  - Key difference: Uses `Service Account` for auth. Preemptible VMs = Spot Instances.  
- **Azure**:  
  - Plugin: **Azure VM Agents**  
  - Uses Azure Resource Manager (ARM) templates. Supports Windows/Linux.  

#### **6. Managing Agent Resources and Health**  
- **Health Checks**:  
  - `Manage Nodes > [Agent] > Log`: View connection issues.  
  - `Agent Status`: `Online`/`Offline`/`Terminated`.  
- **Critical Failures & Fixes**:  
  | **Symptom** | **Cause** | **Solution** |  
  |-------------|-----------|--------------|  
  | `Agent went offline` | Network timeout | Increase `Agent connection timeout` (Manage Jenkins > System) |  
  | `Failed to launch JNLP` | Incorrect secret | Reconnect agent (delete `secret.key` on agent, restart) |  
  | `Disk space low` | Workspace bloat | Enable `Trim excess workspaces` (Agent config) |  
- **Best Practices**:  
  - Set `Retention strategy`: `Idle for [X] minutes` (prevents zombie agents).  
  - Monitor `System Metrics` (Jenkins > Manage Jenkins > System Info) for agent CPU/memory.  
  - Use `Temporary Workspace` (pipeline `agent { docker { } }`) to avoid disk leaks.

---

### **Chapter 14: Distributed Builds and Scalability**  
*Optimizing build throughput across multiple agents.*

#### **1. Load Balancing Builds Across Agents**  
- **How Jenkins Distributes Work**:  
  - Agents with **matching labels** are candidates.  
  - Priority:  
    1. Agents with **fewer builds** (least busy).  
    2. Agents with **exact label match** (vs. expression match).  
- **Tuning Distribution**:  
  - `Manage Jenkins > System > # of executors`: Set per-agent capacity (default=2).  
    - *Rule of thumb*: `# of executors = CPU cores - 1` (e.g., 4-core agent → 3 executors).  
  - **Weighted Load Balancing**: Use `Node and Label Parameter Plugin` to force priority.  

#### **2. Parallel Execution in Pipelines**  
- **Parallel Stages**:  
  ```groovy
  pipeline {
    agent none
    stages {
      stage('Parallel Tests') {
        parallel {
          stage('Linux') {
            agent { label 'linux' }
            steps { sh 'pytest' }
          }
          stage('Windows') {
            agent { label 'windows' }
            steps { bat 'pytest.bat' }
          }
        }
      }
    }
  }
  ```  
- **Dynamic Parallelism**:  
  ```groovy
  def browsers = ['chrome', 'firefox', 'safari']
  stage('Cross-browser') {
    parallel: browsers.collectEntries { browser ->
      ["${browser}" : {
        agent { label 'linux' }
        steps { sh "run-tests.sh ${browser}" }
      }]
    }
  }
  ```  
- **Critical Note**: Parallel stages run *concurrently*, not sequentially → reduces total build time.

#### **3. Dynamic Agent Provisioning**  
- **On-Demand Scaling**:  
  - Cloud agents (EC2/GCE) spin up *only when builds are queued*.  
  - **Provisioning Triggers**:  
    - `Number of executors` > 0 on cloud template.  
    - `Idle Termination` reclaims resources.  
- **Advanced Scaling**:  
  - **EC2 Plugin**: Set `Minimum Instances` = 0, `Maximum Instances` = 10.  
  - **Docker Plugin**: Use `Docker Template Based` with `Pull Strategy: Pull once and share` for faster startup.  

#### **4. Using Jenkins with Cloud Providers**  
- **Best Practices**:  
  - **Spot/Preemptible Instances**: Save costs but handle interruptions:  
    ```groovy
    // In pipeline: Gracefully handle agent termination
    properties([
      durabilityHint('PERFORMANCE_OPTIMIZED'),
      disableConcurrentBuilds()
    ])
    ```  
  - **VPC Peering**: Place Jenkins controller in private subnet. Agents in public subnet (with NAT gateway).  
  - **Security Groups**: Restrict port 50000 to controller's private IP only.  
- **Cost Optimization**:  
  - Use `idleTerminationInMinutes` aggressively (e.g., 5 mins).  
  - Tag cloud instances with `jenkins-agent=true` for cost tracking.

---

### **Chapter 15: Jenkins and Source Control Integration**  
*Automating builds from Git, GitHub, GitLab, and Bitbucket.*

#### **1. Git Integration**  
- **HTTPS vs. SSH**:  
  | **Method** | **Setup** | **Security** |  
  |------------|-----------|--------------|  
  | **HTTPS** | `https://github.com/user/repo.git` | Requires username/token (stored in Jenkins credentials) |  
  | **SSH** | `git@github.com:user/repo.git` | Requires SSH key (Jenkins → Credentials → Add SSH Key) |  
- **Webhooks (Generic)**:  
  1. *Repo Settings*: Add webhook URL: `http://jenkins:8080/github-webhook/` (GitHub) or `http://jenkins:8080/git/notifyCommit` (generic Git).  
  2. *Jenkins Job*: Under `Build Triggers`, check `GitHub hook trigger for GITscm polling` (GitHub) or `Trigger build on push` (Git).  
- **SCM Polling**:  
  - *Jenkins polls repo every X mins* (e.g., `H/5 * * * *` = every 5 mins).  
  - **Downside**: Wastes API calls; slow detection. *Avoid if webhooks work*.

#### **2. GitHub Integration**  
- **Required Plugins**: **GitHub**, **GitHub Branch Source**.  
- **Webhook Setup**:  
  - GitHub Repo → Settings → Webhooks → Add:  
    - Payload URL: `http://jenkins:8080/github-webhook/`  
    - Content type: `application/json`  
    - Events: `Just the push event` (or `Pull Requests` for PR builds).  
- **Pull Request Builds**:  
  - In job config: `GitHub project` URL + `Build when a change is pushed to GitHub`.  
  - For PRs: Check `Build pull requests` (in **GitHub Branch Source** plugin).  
- **Status Checks**: Jenkins posts build status to GitHub PRs (requires `GitHub App` auth for permissions).

#### **3. GitLab CI Integration**  
- **Plugins**: **GitLab**, **GitLab Authentication**.  
- **Webhook**:  
  - GitLab Project → Settings → Webhooks → URL: `http://jenkins:8080/project/my-job`  
  - Trigger: `Push events`, `Merge Request events`.  
- **Pipeline as Code**:  
  - Use `Jenkinsfile` in repo (not `.gitlab-ci.yml`).  
  - GitLab plugin triggers Jenkins on push/PR → Jenkins reads `Jenkinsfile`.

#### **4. Bitbucket Integration**  
- **Plugins**: **Bitbucket**, **Bitbucket Pipeline for Jenkins**.  
- **Webhook**:  
  - Bitbucket → Repository Settings → Webhooks → URL: `http://jenkins:8080/bitbucket-scmsource-hook/notify`  
  - Triggers: `Repository Push`.  
- **PR Builds**:  
  - Enable `Check pull requests` in job config.  
  - Requires Bitbucket Server (Cloud uses different webhook URL).

#### **5. Multi-branch Pipeline Projects**  
- **Purpose**: Auto-detect branches/PRs and build them.  
- **Setup**:  
  1. New Item → `Multibranch Pipeline`.  
  2. Add `Branch Source` (GitHub/GitLab/Bitbucket).  
  3. Jenkins scans repo, creates jobs for:  
     - `main` branch → `main` job  
     - `feature/*` branches → `feature/xyz` job  
     - PRs → `PR-123` job  
- **Jenkinsfile**: Must exist in each branch. Defines pipeline for that branch.  
- **Advanced**:  
  - `Orphaned Item Strategy`: Delete branches inactive for 30 days.  
  - `Periodic Folder Listing`: Scan for new branches every 1h.

#### **6. SCM Polling vs. Webhooks**  
| **Method** | **How It Works** | **Latency** | **Scalability** | **Use Case** |  
|------------|------------------|-------------|-----------------|--------------|  
| **SCM Polling** | Jenkins periodically checks repo (e.g., cron `H/5 * * * *`) | 5+ mins | Poor (high API load) | Legacy repos without webhook support |  
| **Webhooks** | SCM pushes event to Jenkins immediately | < 10 sec | Excellent | Modern CI/CD (GitHub/GitLab/Bitbucket) |  
- **Critical Advice**:  
  > 🚫 **Never use polling in production**. Webhooks are faster, reduce SCM load, and prevent missed commits. If webhooks fail, use `Scan Multibranch Pipeline Triggers` (e.g., daily) as backup.

---

### **Chapter 16: Build Triggers and Automation**  
*Automating when and how builds start.*

#### **1. Polling SCM**  
- **Cron Syntax**:  
  `MINUTE HOUR DOM MONTH DOW`  
  - `H/5 * * * *` = Every 5 mins (staggered to avoid load spikes).  
  - `H 2 * * 1` = Every Monday at 2 AM (staggered hour).  
- **How It Works**:  
  Jenkins checks repo for *new commits* since last build. Triggers build if changes detected.  
- **Pitfall**:  
  High traffic repos → SCM API rate limits (e.g., GitHub: 5,000/hr). Use webhooks instead.

#### **2. Triggering Builds via Webhooks**  
- **Generic Git Webhook**:  
  - URL: `http://jenkins:8080/git/notifyCommit?url=<repo-url>`  
  - *No authentication by default* → **Secure with**:  
    - Firewall (allow only SCM IPs)  
    - Token: `http://jenkins:8080/git/notifyCommit?token=SECRET_TOKEN`  
- **GitHub-Specific**:  
  - Use `GitHub hook trigger` (no token needed if webhook configured correctly).

#### **3. Upstream/Downstream Job Triggers**  
- **Upstream Job**: Job that *triggers* another job.  
- **Downstream Job**: Job that *is triggered*.  
- **Setup**:  
  - **Freestyle**: `Post-build Actions > Build other projects` > `DownstreamJob` (and `Trigger only if build is stable`).  
  - **Pipeline**:  
    ```groovy
    stage('Trigger Downstream') {
      steps {
        build job: 'downstream-job', wait: false, parameters: [string(name: 'BRANCH', value: 'main')]
      }
    }
    ```  
- **Advanced**:  
  - `build` step parameters: `propagate: false` (downstream failure won’t fail upstream).  
  - Use `currentBuild.rawBuild.getCause(hudson.model.Cause$UpstreamCause)` in downstream to get upstream job info.

#### **4. Remote API Trigger (curl, Jenkins CLI)**  
- **Trigger via curl**:  
  ```bash
  curl -X POST http://user:api_token@jenkins:8080/job/my-job/build \
       --data token=SECRET_TOKEN \
       -H "Jenkins-Crumb:$(curl -s 'http://user:api_token@jenkins:8080/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,":",//crumb)')"
  ```  
  - `api_token`: Generated in `User > Configure > API Token`.  
  - `crumb`: Required for CSRF protection (omit if disabled).  
- **Jenkins CLI**:  
  ```bash
  java -jar jenkins-cli.jar -s http://jenkins:8080 -auth user:api_token \
        build "my-job" -p BRANCH=main -v
  ```  
- **Security**: Always use HTTPS and tokens (never passwords in URLs).

#### **5. Timer-Based Builds (cron syntax)**  
- **Use Cases**:  
  - Nightly builds (`H 2 * * *`)  
  - Weekly cleanup (`H 3 * * 0`)  
- **Special Variables**:  
  - `H`: Hashed value (spreads load). `H/15` = every 15 mins, but offset randomly.  
  - `W`: Weekday (e.g., `15 10 * * 1-5` = Mon-Fri at 10:15 AM).  
- **Example Schedules**:  
  | **Schedule** | **Meaning** |  
  |--------------|-------------|  
  | `H H/4 * * *` | Every 4 hours, at a random minute |  
  | `0 9 * * 1-5` | Weekdays at 9:00 AM |  
  | `H(0-29) 18 * * *` | Weekdays between 6:00-6:29 PM |  

#### **6. Parameterized Builds**  
- **Purpose**: Pass inputs to builds (e.g., branch, version).  
- **Setup**:  
  - Job config → `This project is parameterized` → Add parameters:  
    - `String Parameter`: `BRANCH` (default `main`)  
    - `Choice Parameter`: `ENV` (options: `dev, staging, prod`)  
    - `Boolean Parameter`: `RUN_TESTS` (default `true`)  
- **Access in Pipeline**:  
  ```groovy
  pipeline {
    parameters {
      string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch')
      choice(name: 'ENV', choices: ['dev', 'staging', 'prod'])
    }
    stages {
      stage('Deploy') {
        steps {
          sh "deploy.sh --env ${params.ENV} --branch ${params.BRANCH}"
        }
      }
    }
  }
  ```  
- **Triggering with Parameters**:  
  ```bash
  curl -X POST "http://jenkins:8080/job/my-job/buildWithParameters?BRANCH=feature&ENV=staging"
  ```

---

### **Key Takeaways for Production Use**  
1. **Agents**: Always use ephemeral agents for CI (Docker/EC2). Permanent agents only for specialized needs.  
2. **Webhooks > Polling**: Eliminate SCM polling; rely on webhooks for speed and scalability.  
3. **Parallelism**: Use `parallel` in pipelines to cut build times.  
4. **Security**:  
   - Never store SCM credentials in plaintext (use Jenkins Credentials Store).  
   - Restrict webhook IPs (e.g., GitHub IPs: `https://api.github.com/meta`).  
5. **Cost Control**:  
   - Set `idleTermination` aggressively for cloud agents (5-15 mins).  
   - Use Spot/Preemptible instances for non-critical builds.  

> 💡 **Pro Tip**: Use `Jenkins Configuration as Code (JCasC)` to define agents/cloud providers in YAML (avoid UI config drift). Example:  
> ```yaml
> jenkins:
>   nodes:
>     - permanent:
>         name: "linux-builder"
>         remoteFS: "/home/jenkins"
>         labels: "linux docker"
>         launcher:
>           ssh:
>             host: "192.168.1.100"
>             credentialsId: "ssh-cred"
>```
