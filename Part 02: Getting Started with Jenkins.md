### Chapter 3: Installing and Setting Up Jenkins

#### 1. System Requirements
*   **Why it Matters:** Under-provisioning causes slow builds, instability, or failures. Over-provisioning wastes resources.
*   **Absolute Minimum (Testing/Learning ONLY):**
    *   **OS:** Any modern OS (Windows 10+, Ubuntu 20.04+, CentOS 7+, macOS 10.15+)
    *   **CPU:** 1+ Core (2+ recommended)
    *   **RAM:** 1 GB (2+ GB **strongly recommended** for basic usage)
    *   **Disk Space:** 1 GB (10+ GB recommended for builds, plugins, artifacts)
    *   **Java:** Jenkins 2.346+ (LTS) requires **Java 11** (OpenJDK or Oracle JDK). *Older Jenkins versions may need Java 8.* **Jenkins does NOT include Java.**
*   **Recommended for Production (Small-Medium Team):**
    *   **CPU:** 2-4 Cores
    *   **RAM:** 4-8 GB (Scale based on concurrent builds & plugin count)
    *   **Disk Space:** 50-100+ GB (SSD strongly recommended for build workspace & artifact storage)
    *   **Java:** OpenJDK 11 (LTS) or OpenJDK 17 (Jenkins 2.361+ LTS supports it)
*   **Critical Notes:**
    *   **Java is Mandatory:** Install JDK/JRE *before* Jenkins. Verify with `java -version`.
    *   **Disk I/O:** Slow disks cripple build performance. Use SSDs/NVMe.
    *   **Network:** Stable internet for plugin updates. Firewall must allow Jenkins port (default 8080).
    *   **OS Updates:** Keep OS updated for security.
    *   **Jenkins Home (`JENKINS_HOME`):** This directory (default: `/var/lib/jenkins` on Linux, `C:\ProgramData\Jenkins\.jenkins` on Windows) stores *ALL* configuration, jobs, plugins, and build history. **Ensure it has ample, fast, and reliable storage. Back it up regularly!**

#### 2. Installing Jenkins on Windows
*   **Method 1: Windows Installer (Recommended for Simplicity)**
    1.  **Download:** Go to [https://www.jenkins.io/download/](https://www.jenkins.io/download/), select "Long-Term Support Release" > "Windows" > Download the `.msi` file.
    2.  **Run Installer:** Double-click the `.msi` file. Run as Administrator.
    3.  **Installation Steps:**
        *   Accept License Agreement.
        *   **Choose Installation Directory:** Default is `C:\Program Files\Jenkins`. Change if needed (ensure path has no spaces if possible, though modern Jenkins handles them).
        *   **Service Account:** Choose the user account Jenkins service runs as. **Best Practice:** Create a dedicated local user (e.g., `jenkins`) with minimal privileges. *Do NOT use `Local System` for production security.*
        *   **Port:** Default is `8080`. Change if needed (e.g., `8081` if port 8080 is busy). Ensure firewall allows this port.
        *   **Run Service:** Select "Run service as the logged-in user" only for testing. For production, use the dedicated service account chosen above.
        *   Complete installation.
    4.  **Verify Service:** Open `Services.msc`. Look for "Jenkins" service. It should be running. Right-click > Properties to check account and startup type (should be `Automatic`).
*   **Method 2: WAR File (Manual - More Control)**
    1.  **Download:** Get the `jenkins.war` file from [https://www.jenkins.io/download/](https://www.jenkins.io/download/).
    2.  **Open Command Prompt:** Run as Administrator.
    3.  **Navigate & Run:**
        ```cmd
        cd C:\path\to\downloaded\file
        java -jar jenkins.war --httpPort=8080
        ```
        *   Replace `8080` with your desired port.
        *   **Warning:** This runs Jenkins in the foreground. Closing the prompt stops Jenkins. Use for testing only.
    4.  **(Optional) Create Service:** For production, create a Windows Service using `nssm` (Non-Sucking Service Manager) or `winsw` to run the WAR file persistently.
*   **Post-Install (Both Methods):**
    *   Jenkins home is typically `C:\ProgramData\Jenkins\.jenkins` (hidden folder).
    *   Access Jenkins at `http://localhost:8080` (or your chosen port).

#### 3. Installing Jenkins on Linux (Ubuntu, CentOS)
*   **Prerequisite: Install Java (OpenJDK 11)**
    *   **Ubuntu:**
        ```bash
        sudo apt update
        sudo apt install -y openjdk-11-jdk
        java -version # Verify (should show openjdk 11.x.x)
        ```
    *   **CentOS/RHEL:**
        ```bash
        sudo yum install -y java-11-openjdk-devel
        java -version # Verify
        ```
*   **Method 1: Package Repository (Recommended - Automatic Updates)**
    *   **Ubuntu:**
        1.  Add Jenkins repository key:
            ```bash
            curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
              /usr/share/keyrings/jenkins-keyring.asc > /dev/null
            ```
        2.  Add repository to sources list:
            ```bash
            echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
              https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
              /etc/apt/sources.list.d/jenkins.list > /dev/null
            ```
        3.  Install Jenkins:
            ```bash
            sudo apt update
            sudo apt install -y jenkins
            ```
    *   **CentOS/RHEL:**
        1.  Add Jenkins repository:
            ```bash
            sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
            sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
            ```
        2.  Install Jenkins:
            ```bash
            sudo yum install -y jenkins
            ```
*   **Method 2: Manual WAR File (Less Common on Linux)**
    1.  Download `jenkins.war` (as on Windows).
    2.  Run: `java -jar jenkins.war --httpPort=8080` (Foreground, for testing).
    3.  **(Production)** Create a systemd service (see below).
*   **Starting & Managing Jenkins (Package Install):**
    *   **Start:** `sudo systemctl start jenkins`
    *   **Enable on Boot:** `sudo systemctl enable jenkins`
    *   **Status:** `sudo systemctl status jenkins`
    *   **Restart:** `sudo systemctl restart jenkins`
    *   **Logs:** `sudo journalctl -u jenkins -f` (Follow logs)
*   **Firewall Configuration:**
    *   **Ubuntu (UFW):** `sudo ufw allow 8080/tcp`
    *   **CentOS (firewalld):**
        ```bash
        sudo firewall-cmd --permanent --add-port=8080/tcp
        sudo firewall-cmd --reload
        ```
*   **Jenkins Home:** `/var/lib/jenkins` (Contains `jobs/`, `plugins/`, `secrets/`, `config.xml`, `users/`, etc.)
*   **(Critical) SELinux (CentOS/RHEL):** Jenkins might be blocked. Temporarily set to permissive to test: `sudo setenforce 0`. If Jenkins works, make a permanent policy:
    ```bash
    sudo semanage port -a -t http_port_t -p tcp 8080  # Allows Jenkins port
    sudo setsebool -P httpd_can_network_connect 1     # If Jenkins needs network access
    ```

#### 4. Installing Jenkins on macOS
*   **Prerequisite: Install Java (OpenJDK 11)**
    *   Best: Use [Homebrew](https://brew.sh/): `brew install openjdk@11`
    *   Or download from [Adoptium](https://adoptium.net/).
    *   Verify: `java -version`
*   **Method 1: Homebrew (Recommended)**
    1.  Install Homebrew (if not installed): `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
    2.  Install Jenkins LTS:
        ```bash
        brew install jenkins-lts
        ```
    3.  **Start Jenkins:**
        *   **Temporary (Foreground):** `brew services start jenkins-lts` (Runs until you stop it)
        *   **Persistent (Background):** `brew services start jenkins-lts` (Starts on login/boot)
    4.  **Manage:**
        *   Stop: `brew services stop jenkins-lts`
        *   Restart: `brew services restart jenkins-lts`
*   **Method 2: Installer Package (jenkins.io)**
    1.  Download the macOS `.pkg` installer from [https://www.jenkins.io/download/](https://www.jenkins.io/download/).
    2.  Run the installer. It will:
        *   Install Jenkins as a launch daemon (runs on system boot).
        *   Set up Jenkins home in `/Users/Shared/Jenkins/Home`.
        *   Configure a dedicated `jenkins` user.
        *   Open port `8080`.
    3.  Start/Stop: Managed automatically via launchd. Use `sudo launchctl list | grep jenkins` to check status.
*   **Jenkins Home:** `/Users/Shared/Jenkins/Home` (Method 2) or `$(brew --prefix)/var/jenkins_home` (Method 1 - usually `/usr/local/var/jenkins_home`).
*   **Access:** `http://localhost:8080`

#### 5. Installing Jenkins via Docker
*   **Why Docker?** Isolated, consistent environment, easy version switching, good for testing/demo.
*   **Critical:** **Persist Jenkins Home!** Docker containers are ephemeral. Without volume mounts, *all data is lost on container restart*.
*   **Basic Run Command (Linux/macOS):**
    ```bash
    docker run -d -p 8080:8080 -p 50000:50000 --name jenkins \
      -v jenkins-data:/var/jenkins_home \
      -v /var/run/docker.sock:/var/run/docker.sock \  # Optional: Allows Jenkins to run Docker-in-Docker
      jenkins/jenkins:lts
    ```
    *   `-d`: Run detached (background).
    *   `-p 8080:8080`: Map host port 8080 to container port 8080 (web UI).
    *   `-p 50000:50000`: Map agent port (for connecting build agents).
    *   `--name jenkins`: Name the container.
    *   `-v jenkins-data:/var/jenkins_home`: **CRITICAL.** Creates a named Docker volume `jenkins-data` to persist Jenkins home. *Never omit this.*
    *   `-v /var/run/docker.sock...`: Optional but common. Allows Jenkins to launch Docker containers for builds (requires Docker-in-Docker setup).
    *   `jenkins/jenkins:lts`: Official Jenkins LTS image.
*   **Windows (PowerShell):**
    ```powershell
    docker run -d -p 8080:8080 -p 50000:5000 --name jenkins `
      -v jenkins-data:C:\var\jenkins_home `  # Windows path syntax differs!
      jenkins/jenkins:lts
    ```
    *   **Important:** Windows volume paths use backslashes or forward slashes within quotes, but Docker expects Linux-style paths *inside* the container. The volume mount `-v jenkins-data:C:\var\jenkins_home` tells Docker to map the volume to `C:\var\jenkins_home` *within the Windows container*. The official Windows image uses `C:\var\jenkins_home`.
*   **Finding Initial Admin Password:**
    ```bash
    docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
    # Windows Container: docker exec jenkins type C:\var\jenkins_home\secrets\initialAdminPassword
    ```
*   **Managing the Container:**
    *   View Logs: `docker logs jenkins`
    *   Stop: `docker stop jenkins`
    *   Start: `docker start jenkins`
    *   Remove (Data preserved in volume!): `docker rm jenkins`
    *   **Upgrade:** `docker pull jenkins/jenkins:lts` then `docker stop jenkins` + `docker rm jenkins` + run new `docker run` command. Volume `jenkins-data` persists config/plugins/jobs.
*   **Volume Location:** Docker stores named volumes (`jenkins-data`) in its internal storage (e.g., `/var/lib/docker/volumes/` on Linux). Back up using `docker run --rm -v jenkins-data:/volume -v /backup:/backup alpine tar cvf /backup/jenkins-backup.tar -C /volume .`

#### 6. Installing Jenkins on Kubernetes
*   **Why Kubernetes?** High availability, scalability, resilience, integrates with cloud-native ecosystems.
*   **Recommended Method: Helm Chart (Simplest)**
    1.  **Prerequisites:**
        *   Kubernetes cluster (v1.20+ recommended)
        *   `kubectl` configured
        *   Helm v3 installed
        *   StorageClass configured for persistent volumes (PVs)
    2.  **Add Helm Repo:**
        ```bash
        helm repo add jenkins https://charts.jenkins.io
        helm repo update
        ```
    3.  **Create Custom Values File (`jenkins-values.yaml`):** **CRITICAL STEP - Do NOT use defaults for production!**
        ```yaml
        controller:
          # Admin credentials (CHANGE THESE!)
          adminUser: "admin"
          adminPassword: "your_strong_password_here" # Or use existingSecret for better security
          # Jenkins Home Size & Storage
          persistence:
            enabled: true
            size: "50Gi" # Adjust based on needs
            storageClass: "your-preferred-storage-class" # e.g., "standard" on GKE
          # Resources (Adjust based on load)
          resources:
            requests:
              memory: "2Gi"
              cpu: "500m"
            limits:
              memory: "4Gi"
              cpu: "1000m"
          # Service (How to access Jenkins)
          serviceType: "ClusterIP" # Use Ingress or NodePort/LB externally
          # Ingress (Recommended for external access)
          ingress:
            enabled: true
            hostName: "jenkins.yourdomain.com"
            tls: true
            tlsSecret: "jenkins-tls-secret" # Pre-created TLS secret
          # Java Options (Tune memory)
          javaOpts: "-Xmx2048m -Xms512m"
        ```
    4.  **Install Jenkins:**
        ```bash
        helm upgrade --install my-jenkins jenkins/jenkins -f jenkins-values.yaml --namespace jenkins --create-namespace
        ```
        *   `my-jenkins`: Release name.
        *   `--namespace jenkins`: Creates and uses a dedicated namespace.
    5.  **Access:**
        *   **Via Ingress:** `https://jenkins.yourdomain.com` (Requires DNS setup & TLS cert).
        *   **Via Port-Forward (Testing):** `kubectl -n jenkins port-forward svc/my-jenkins-controller 8080:8080`
*   **Key Considerations for K8s:**
    *   **Persistence is Non-Negotiable:** Configure `controller.persistence` correctly. Losing the PV means losing *everything*.
    *   **Resource Requests/Limits:** Essential for cluster stability and predictable performance.
    *   **Security:**
        *   **Never** use default admin credentials. Set strong `adminPassword` or use `existingSecret`.
        *   **Always** use TLS (Ingress with valid cert).
        *   Restrict network policies.
        *   Run Jenkins controller with non-root user (Helm chart supports this via `controller.securityContext`).
    *   **Scalability:** Configure `controller.numExecutors` and consider using [Kubernetes Plugin](https://plugins.jenkins.io/kubernetes/) for dynamic agent provisioning (pods spun up per build).
    *   **Backup:** Back up the Persistent Volume Claim (PVC) containing `JENKINS_HOME`. Tools like Velero are ideal.

#### 7. Accessing Jenkins Web UI
*   **URL:** `http://<your-jenkins-server-ip-or-hostname>:<port>` (e.g., `http://localhost:8080`, `http://jenkins.example.com:8080`)
*   **First Access:** Triggers the **Initial Setup Wizard** (see next section).
*   **Common Issues:**
    *   **Connection Refused:** Jenkins not running, firewall blocking port, wrong port specified.
    *   **404 Not Found:** Incorrect context path (Jenkins might be behind a reverse proxy with a path like `/jenkins` - check proxy config).
    *   **502 Bad Gateway:** Reverse proxy (Nginx, Apache) configured but Jenkins not running or unreachable.
*   **Troubleshooting:**
    1.  Check Jenkins service status (`systemctl status jenkins`, `docker logs jenkins`, `kubectl -n jenkins get pods`).
    2.  Verify port is listening (`netstat -tuln | grep 8080`).
    3.  Check firewall settings.
    4.  Check Jenkins logs (`/var/log/jenkins/jenkins.log`, Docker logs, K8s pod logs).

#### 8. Initial Setup Wizard (Unlock Jenkins, Plugin Installation, Admin User Creation)
*   **Step 1: Unlock Jenkins**
    *   **Why:** Security measure to prevent unauthorized initial setup.
    *   **Location:** First screen you see after accessing the URL.
    *   **Finding the Password:**
        *   **Linux (Package):** `sudo cat /var/lib/jenkins/secrets/initialAdminPassword`
        *   **Windows (Installer):** `C:\ProgramData\Jenkins\.jenkins\secrets\initialAdminPassword`
        *   **Docker:** `docker exec <container-name> cat /var/jenkins_home/secrets/initialAdminPassword`
        *   **Kubernetes:** `kubectl -n jenkins exec <jenkins-pod-name> -- cat /var/jenkins_home/secrets/initialAdminPassword`
        *   **macOS (Homebrew):** `cat $(brew --prefix)/var/jenkins_home/secrets/initialAdminPassword`
    *   **Action:** Copy the password, paste it into the field, click "Continue".
*   **Step 2: Customize Jenkins (Plugin Installation)**
    *   **Options:**
        *   **Install suggested plugins:** **RECOMMENDED for beginners.** Installs a curated set of essential plugins (Git, Pipeline, GitHub, Credentials Binding, etc.). Good starting point.
        *   **Select plugins to install:** Advanced option. Only choose if you know exactly which minimal plugins you need (e.g., for extreme resource constraints). Risky - might miss critical plugins.
    *   **What Happens:** Jenkins downloads and installs the selected plugins from the update center. Progress shown on screen. *This can take several minutes depending on network speed.*
    *   **Troubleshooting Plugin Install:**
        *   **Stuck/Slow:** Check network connectivity from Jenkins server to `updates.jenkins.io`. Firewall/proxy might block it. Configure proxy in `Manage Jenkins > Manage Plugins > Advanced` *after* initial setup (requires restarting setup).
        *   **Plugin Errors:** Note the failing plugin. Retry later (transient issue) or investigate specific plugin compatibility.
*   **Step 3: Create First Admin User**
    *   **Why:** The initial admin account created here is your primary superuser account. **Do not skip this!**
    *   **Fields:**
        *   **Full name:** Your name or team name (e.g., "DevOps Team").
        *   **Username:** **CRITICAL.** Choose wisely! This is your primary login ID. *Avoid "admin" for security.* (e.g., `jenkins-admin`, `alice_devops`). Cannot be easily changed later.
        *   **Password:** **STRONG PASSWORD REQUIRED.** Use a unique, complex password (12+ chars, mix upper/lower/number/symbol). *Never reuse passwords.*
        *   **Confirm password:** Type it again.
        *   **E-mail address:** Optional but recommended for notifications.
    *   **Action:** Click "Save and Continue".
*   **Step 4: Instance Configuration**
    *   **Jenkins URL:** The base URL clients (like git servers) will use to connect *back* to Jenkins (e.g., for webhooks). **Must be correct!**
        *   **Default:** Usually `http://<your-server>:8080/`. If behind a reverse proxy (Nginx/Apache) using HTTPS and a domain (e.g., `https://jenkins.example.com`), **you MUST change this to the external URL (`https://jenkins.example.com/`)**. Incorrect URL breaks webhooks and agent connections.
    *   **Action:** Verify/Correct the URL, click "Save and Finish".
*   **Step 5: Start Using Jenkins**
    *   Click "Start using Jenkins".
    *   You are now logged in as your new admin user on the Jenkins Dashboard.

---

### Chapter 4: Jenkins Dashboard & Basic Navigation

#### 1. Jenkins Dashboard Overview
*   **The Central Hub:** Your starting point for almost all Jenkins activities. Accessed via the top-left "Jenkins" logo or the root URL.
*   **Key Sections:**
    *   **Top Navigation Bar:**
        *   **Jenkins Logo:** Link back to Dashboard.
        *   **Search Box:** Quickly find jobs by name.
        *   **User Icon:** Profile, credentials, logout.
        *   **+ Menu:** "New Item" (Create Job/Pipeline), "New View" (Organize jobs).
        *   **(Admin) Manage Jenkins:** Global configuration (covered later).
    *   **Main Content Area (Default View - "All"):**
        *   **Job List:** Shows all top-level jobs (Freestyle, Pipeline, etc.) in the default view ("All").
            *   **Job Name:** Click to open the job.
            *   **Status Icon:** (Blue ball = success, Red = failure, Yellow = unstable, Grey = disabled/aborted, Spinny ball = building).
            *   **Last Build:** Number and status of the most recent build.
            *   **Build Graph:** Mini trend graph showing recent build statuses (success/fail).
            *   **Actions:** (Gear icon) Quick links: Build Now, Configure, Delete, etc.
        *   **Views Menu (Top Left):** Switch between different job organization views (e.g., "All", "My Views", custom views).
    *   **System Messages (Below Jobs):** Critical alerts (e.g., "Jenkins is about to shut down", "Update available", "Disk space low").
*   **Understanding "Views":** Views are filters/organizers for jobs. The default "All" view shows every job. You can create custom views (e.g., "Production Jobs", "Team A Jobs", "Failed Jobs") to group relevant jobs together.

#### 2. Managing Jobs and Pipelines
*   **Job vs. Pipeline:**
    *   **Job:** Generic term for a configured task in Jenkins (Freestyle Project, Pipeline, Multibranch Pipeline, etc.).
    *   **Freestyle Project:** Traditional, GUI-configured job. Good for simple, single-repo tasks. Less flexible for complex workflows.
    *   **Pipeline (Declarative/Scripted):** Defined primarily via code (Jenkinsfile) stored in source control. **Modern standard.** Enables complex workflows, stages, parallelism, and "Pipeline as Code". More maintainable and auditable.
*   **Creating a Job/Pipeline:**
    1.  Click "New Item" (top left `+` menu or Dashboard sidebar).
    2.  Enter **Item name** (Unique name, no spaces/special chars recommended).
    3.  Select **type**:
        *   `Freestyle project`
        *   `Pipeline` (For a single-pipeline job)
        *   `Multibranch Pipeline` (For auto-discovering pipelines in branches/tags of a repo - **highly recommended for Git workflows**)
        *   (Other types: Maven, GitHub Organization, etc.)
    4.  Click "OK".
*   **Configuring a Job:**
    *   **Freestyle Project:** Extensive GUI forms (Source Code Management, Build Triggers, Build Environment, Build Steps, Post-build Actions).
    *   **Pipeline:**
        *   **Definition:** Choose `Pipeline script` (for testing) or **`Pipeline script from SCM` (Production Standard)**. This points to a `Jenkinsfile` in your source code repository.
        *   **SCM Configuration:** Specify repo URL, credentials, branch to scan (for `Pipeline` type) or branch sources (for `Multibranch Pipeline`).
    *   **Essential Settings (Both):**
        *   **Description:** Document what the job does.
        *   **Discard Old Builds:** **CRITICAL.** Configure to keep only last X builds or builds from last Y days to save disk space. (e.g., "Max # of builds to keep: 20", "Days to keep builds: 30").
        *   **Parameters:** Make jobs reusable (e.g., String, Choice, Boolean parameters).
*   **Running a Job:**
    *   **Manual:** On job page, click "Build Now" (sidebar) or from Dashboard action menu (gear icon).
    *   **Automated:** Via triggers (SCM polling, webhooks, schedule (cron), upstream job completion).
*   **Viewing a Build:**
    *   Click the build number in the job's build history.
    *   **Key Tabs:**
        *   **Console Output:** Real-time/build log. **Primary place for debugging failures.**
        *   **Changes:** Commits included in this build (if SCM configured).
        *   **Tests:** Test results (if published).
        *   **Artifacts:** Files archived by the build (if configured).
        *   **Pipeline Steps:** (For Pipelines) Visual breakdown of each stage/step.
*   **Managing Jobs:**
    *   **Rename:** Job Config > Advanced Project Options > "Rename".
    *   **Copy:** Job Config > "Copy this job".
    *   **Delete:** Job Page > Sidebar > "Delete" (Confirm carefully!).
    *   **Disable/Enable:** Job Page > Sidebar > "Disable"/"Enable". Stops builds but keeps config/history.

#### 3. Understanding the Sidebar Menu
*   **The Action Hub:** Context-specific actions based on where you are (Dashboard, Job Page, etc.).
*   **Dashboard Sidebar:**
    *   **Status:** System health summary (nodes, executors, queue).
    *   **People:** List of configured users.
    *   **Build Queue:** Currently queued builds waiting for executors.
    *   **Build Executor Status:** Real-time view of all executors (on controller & agents) and what they are running.
    *   **Manage Jenkins:** **Global Configuration** (See next section).
    *   **New Item:** Create a new job.
    *   **New View:** Create a new job organization view.
    *   **Credentials:** Manage stored credentials (usernames, tokens, SSH keys, certs). *Crucial for security.*
    *   **(Logged in User) Log out:** Self-explanatory.
*   **Job Page Sidebar:**
    *   **Workspace:** View files in the job's build workspace on the agent (useful for debugging).
    *   **Build Now:** Trigger a manual build.
    *   **(Build Running) Cancel Build:** Stop a running build.
    *   **Configure:** Edit the job's configuration.
    *   **Delete:** Remove the job.
    *   **Enable/Disable:** Toggle job status.
    *   **Rename:** Change job name.
    *   **Copy:** Duplicate the job.
    *   **(Pipeline Jobs) Scan Multibranch Pipeline Now:** (For Multibranch) Trigger branch scanning.
    *   **(Pipeline Jobs) Replay:** (Declarative Pipeline) Rerun a specific build with modified Jenkinsfile (great for debugging).

#### 4. Global Configuration Settings (`Manage Jenkins`)
*   **The Heart of Jenkins:** Where you configure system-wide behavior. **Access via Sidebar > Manage Jenkins.**
*   **Critical Sections:**
    *   **Configure System:**
        *   **Jenkins Location:** **MOST IMPORTANT.** `Jenkins URL` must match the external URL users/clients use (e.g., `https://jenkins.yourcompany.com/`). Incorrect URL breaks webhooks and agent connections.
        *   **Email Notification:** SMTP server settings for sending build result emails.
        *   **GitHub/GitLab:** Configure connections to your Git servers (API endpoints, credentials).
        *   **Maven/Gradle/Ant:** Configure global installations of build tools (if not using tool auto-install in jobs).
        *   **Clouds:** Configure dynamic agent provisioning (Kubernetes, EC2, etc.).
        *   **Global properties:** Environment variables applied to *all* jobs.
    *   **Manage Plugins:**
        *   **Installed:** View/update/remove plugins. **Keep plugins updated!** (Check "Security Warnings" tab).
        *   **Available:** Browse/search for new plugins. Install manually or select for bulk install.
        *   **Updates:** Check for and install updates to *all* plugins at once (do this regularly!).
        *   **Advanced:** Configure update center URL (e.g., for air-gapped networks), proxy settings.
    *   **Security Realm:**
        *   **Login:** Configure *how* users authenticate.
            *   `Jenkins’ own user database` (Default - manage users internally).
            *   `Unix user/group database` (Linux only).
            *   **`LDAP` / `Active Directory` (Common for enterprises).**
            *   `GitHub Authentication Plugin`, `GitLab Authentication Plugin`, `SAML`, `OAuth` (Integrate with identity providers).
    *   **Authorization:**
        *   **Who can do what?** **CRITICAL FOR SECURITY.**
            *   `Logged-in users can do anything` (Default - **INSECURE! Avoid in production**).
            *   `Matrix-Based Security` (Granular control per user/group: Administer, Job/Run/View/Create/Configure/Delete permissions). **Recommended starting point.**
            *   `Project-based Matrix Authorization Strategy` (Fine-grained control per job). Use with Matrix security.
            *   `Role-Based Strategy` (Advanced - define roles like "Developer", "Admin", assign permissions to roles, assign users to roles). **Best for larger teams.**
        *   **Always create at least one non-admin user for daily tasks!** Never use the initial admin account for routine work.
    *   **Configure Global Security:** (Often combines Security Realm & Authorization settings).
    *   **Manage Credentials:**
        *   **System:** Credentials used by Jenkins itself (e.g., for cloning repos, deploying).
            *   `System` > `Global credentials (unrestricted)`: Credentials available to all jobs. **Use sparingly!**
            *   `System` > `Domain` (e.g., `github.com`): Credentials scoped to a specific domain (more secure).
        *   **Stores:** `System` (Jenkins home) is the default. Can add others (e.g., HashiCorp Vault).
        *   **Types:** Username/Password, SSH Username with private key, Secret text (API tokens), Certificate, Amazon Web Services credentials. **NEVER store secrets in job configs! Use Credentials.**
    *   **Manage Nodes and Clouds:**
        *   **Nodes:** Configure static build agents (permanent agents connected directly to controller).
        *   **Clouds:** Configure dynamic cloud providers (Kubernetes, AWS EC2, Azure, OpenStack) to spin up agents on demand. **Preferred method for scalability.**
    *   **Script Console:** **EXTREME CAUTION.** Groovy script execution with full admin privileges. **Only for experts in emergencies.** Misuse can destroy Jenkins instance.
    *   **System Log:** View Jenkins controller logs (`/var/log/jenkins/jenkins.log` equivalent). Essential for debugging startup/plugin issues.
    *   **Plugins:** (Shortcut to Manage Plugins).
    *   **Global Tool Configuration:** Define global installations of JDK, Maven, Gradle, NodeJS, etc. Jobs can then "refer" to these named installations instead of hardcoding paths.

#### 5. Managing Users and Security
*   **Creating Users:**
    *   `Manage Jenkins` > `Manage Users` > `Create User`.
    *   Fill in details (Username, Password, Full Name, E-mail). **Enforce strong passwords.**
*   **User Management:**
    *   `Manage Jenkins` > `Manage Users`: View, disable, delete users. Reset passwords.
*   **Authentication (How users log in):** Configured in `Manage Jenkins` > `Configure Global Security` > `Security Realm` (See Section 4 above).
*   **Authorization (What users can do):** Configured in `Manage Jenkins` > `Configure Global Security` > `Authorization` (See Section 4 above - Matrix/Role-based).
    *   **Best Practice Workflow:**
        1.  Set up Authentication (e.g., LDAP/AD).
        2.  Switch Authorization to `Matrix-Based Security` or `Role-Based Strategy`.
        3.  Add your admin user and grant `Administer` permission.
        4.  Add other users/groups and grant *least privilege necessary* (e.g., `Job > Read`, `Job > Build`, `View > Read`). **Do NOT grant `Administer` unless absolutely required.**
        5.  (Role-Based) Create roles (e.g., "Developer", "Deployer", "Admin"), assign permissions to roles, assign users to roles.
*   **Credentials Management:** (See Section 4 - `Manage Credentials`)
    *   **Principle of Least Privilege:** Create credentials with *only* the permissions needed for the task (e.g., a read-only deploy key for cloning, a deploy token with only `write` to a specific registry).
    *   **Scope:** Prefer `Domain` scoped credentials over `Global` whenever possible.
    *   **Rotation:** Have a process to rotate credentials (API tokens, passwords) periodically.
*   **Critical Security Hardening:**
    *   **HTTPS:** **MANDATORY.** Terminate TLS at a reverse proxy (Nginx, Apache, Cloud LB) in front of Jenkins. Never expose HTTP (port 8080) directly to the internet.
    *   **Firewall:** Restrict access to Jenkins port (8080/8443) to trusted networks/IPs (e.g., only your corporate network or CI/CD agents).
    *   **Agent -> Controller Security:** Use the "Agent -> Controller" access control (in `Configure Global Security`) to restrict what agents can request from the controller (prevents malicious agents from stealing secrets).
    *   **Plugin Hygiene:** Only install necessary plugins. Remove unused plugins. Keep plugins updated. Monitor [Jenkins Security Advisories](https://www.jenkins.io/security/advisories/).
    *   **Backup:** Regularly back up `JENKINS_HOME` (includes `config.xml`, `secrets/`, `users/`, `jobs/`, `plugins/`). Test restores!
    *   **Audit:** Enable auditing (plugins like `audit-trail`) to track critical configuration changes and job executions.

#### 6. Monitoring System Status
*   **Why:** Proactive monitoring prevents outages, identifies bottlenecks, ensures healthy builds.
*   **Key Areas & How to Check:**
    *   **Build Queue (`Dashboard Sidebar > Build Queue`):**
        *   **Problem:** Long queue = not enough executors. Builds wait unnecessarily.
        *   **Fix:** Add more static agents (`Manage Nodes`) or configure Clouds for dynamic agents. Increase executors per node.
    *   **Executor Status (`Dashboard Sidebar > Build Executor Status`):**
        *   **Problem:** Executors consistently busy (red) or idle (grey) when builds are queued.
        *   **Fix:** Scale agents up/down based on demand (Clouds are ideal for this).
    *   **Disk Space (`Manage Jenkins > System Log` or OS monitoring):**
        *   **Problem:** `JENKINS_HOME` filling up (especially `jobs/<job>/builds/`). Causes build failures.
        *   **Fix:** Configure `Discard Old Builds` aggressively in jobs. Clean up manually (`rm -rf jobs/*/builds/<old-builds>` - **BE CAREFUL!**). Increase disk space.
    *   **Memory/CPU Usage (OS Monitoring - `top`, `htop`, `docker stats`, K8s metrics):**
        *   **Problem:** Jenkins controller or agent using excessive resources, causing slowness or OOM kills.
        *   **Fix:** Tune JVM heap (`javaOpts` in service config/Docker/K8s). Scale controller resources. Optimize pipelines (avoid huge workspaces, long-running steps).
    *   **Plugin Updates (`Manage Jenkins > Manage Plugins > Updates`):**
        *   **Problem:** Outdated plugins with security vulnerabilities or bugs.
        *   **Fix:** Schedule regular plugin update windows. Test updates in a staging environment first if possible.
    *   **System Log (`Manage Jenkins > System Log`):**
        *   **Problem:** Errors, warnings, stack traces indicating misconfiguration, plugin conflicts, or resource issues.
        *   **Fix:** Read the log! It's the primary source for diagnosing *why* something is failing. Search online for specific error messages.
    *   **Jenkins Health Checks (Advanced):**
        *   Use plugins like `prometheus` to expose metrics to monitoring systems (Prometheus/Grafana).
        *   Set up alerts for: High queue length, disk space > 80%, plugin update available (security), controller restarts.
*   **Proactive Monitoring Mindset:** Don't wait for builds to fail. Check queue, executors, and disk space daily (or set up automated alerts).

---
