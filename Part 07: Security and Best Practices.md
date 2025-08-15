### **Chapter 20: Jenkins Security**

#### **1. Authentication Methods**
*   **Jenkins' Own User Database (Internal Database):**
    *   **How it Works:** Users created/stored directly in `$JENKINS_HOME/users/`. Passwords hashed with bcrypt (configurable strength).
    *   **Configuration:** `Manage Jenkins > Security > Access Control > Security Realm > Jenkins’ own user database`. Enable "Allow users to sign up" cautiously.
    *   **Pros:** Simple setup, no external dependency.
    *   **Cons:** **Major Risk:** Password reset requires admin access (lost if admin account compromised). No SSO. Unsuitable for large teams. **Critical:** Disable self-registration in production.
    *   **Best Practice:** Only for small teams or initial setup. **Always enable "Prevent users from creating accounts" in production.** Use only as fallback if external auth fails.

*   **LDAP (Lightweight Directory Access Protocol):**
    *   **How it Works:** Jenkins binds to LDAP server (Active Directory, OpenLDAP) to validate credentials and fetch group membership.
    *   **Configuration:** `Security Realm > LDAP`. Key settings:
        *   `Server`: `ldap://your-ldap-server:389` or `ldaps://...:636` (use LDAPS!).
        *   `Root DN`: `dc=example,dc=com`
        *   `User Search Base`: `ou=users`
        *   `User Search Filter`: `(sAMAccountName={0})` (AD) or `(uid={0})` (OpenLDAP)
        *   `Group Search Base`: `ou=groups`
        *   `Group Search Filter`: `(member={0})` (AD) or `(uniqueMember={0})` (OpenLDAP)
        *   **Bind DN/Password:** Service account with *read-only* access to user/group info (e.g., `cn=jenkins-svc,ou=service-accounts,dc=example,dc=com`).
        *   **Enable TLS/SSL:** **MANDATORY** for production. Use `ldaps://` or STARTTLS.
    *   **Pros:** Centralized user/group management. Leverages existing corporate identity.
    *   **Cons:** Complex configuration. Group sync can be slow. Requires network access to LDAP.
    *   **Critical Tips:**
        *   **ALWAYS use LDAPS (port 636) or STARTTLS.** Never plain LDAP over untrusted networks.
        *   Test connection with `ldapsearch` CLI *before* configuring Jenkins.
        *   Configure `Group Membership Strategy` carefully (e.g., "From user record attribute" for nested groups in AD).
        *   Set `Cache size` (e.g., 100) and `Cache TTL` (e.g., 300 seconds) to reduce LDAP load.

*   **SSO (Single Sign-On): SAML & OAuth/OpenID Connect**
    *   **SAML (Security Assertion Markup Language):**
        *   **How it Works:** Jenkins acts as Service Provider (SP). Identity Provider (IdP - Okta, Azure AD, Auth0) authenticates user and sends SAML assertion.
        *   **Configuration:** Requires **SAML Plugin**. `Security Realm > SAML 2.0`.
            *   `Identity Provider Metadata`: URL or XML file from IdP.
            *   `Relay State`: Usually empty.
            *   `Username Attribute`: `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name` (AD FS) or `email` (Okta).
            *   `Groups Attribute`: `http://schemas.xmlsoap.org/claims/Group` (AD FS) or `groups` (Okta) - **CRITICAL for authorization**.
            *   `Display Name Attribute`: `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname`
            *   `Logout URL`: IdP logout endpoint.
        *   **Critical Steps:**
            1.  Configure Jenkins as SP in IdP (ACS URL: `http(s)://<jenkins-url>/securityRealm/finishLogin`).
            2.  Map IdP groups to Jenkins roles (see Authorization section).
            3.  **Enable "Force users to re-authenticate for sensitive operations"** in plugin settings.
        *   **Pros:** True enterprise SSO. Strong security. Centralized session management.
        *   **Cons:** Complex setup. IdP dependency. Session timeouts controlled by IdP.
    *   **OAuth/OpenID Connect (e.g., GitHub, Google, Azure AD):**
        *   **How it Works:** Jenkins uses OAuth2 flow. User redirected to OAuth provider (e.g., GitHub), grants consent, provider sends access token & ID token (OpenID Connect) back to Jenkins.
        *   **Configuration:** Requires **OAuth Plugin** or **Azure AD Plugin** etc. `Security Realm > [Provider Name]`.
            *   `Client ID` / `Client Secret`: From provider's app registration.
            *   `Authorization Server URL` / `Token Server URL`: Provider-specific (e.g., GitHub: `https://github.com/login/oauth/authorize`, `https://github.com/login/oauth/access_token`).
            *   `UserInfo URL`: `https://api.github.com/user` (GitHub)
            *   `Token Info URL`: Often same as UserInfo URL for OIDC.
            *   `Scopes`: `read:user` (GitHub), `openid profile email` (OIDC).
            *   `Username Field`: `login` (GitHub), `preferred_username` (OIDC).
            *   `Groups Field`: `organizations` (GitHub - requires `read:org` scope), `groups` (OIDC - requires group claim).
        *   **Critical Steps:**
            1.  Register Jenkins as OAuth App in provider (Callback URL: `http(s)://<jenkins-url>/securityRealm/finishLogin`).
            2.  Configure group mapping if needed (GitHub orgs/repos, OIDC groups claim).
        *   **Pros:** Seamless for developer-centric tools (GitHub/GitLab). Modern standard.
        *   **Cons:** Provider-specific quirks. Group mapping often less robust than SAML/LDAP. Revocation requires provider action.

*   **Authentication Best Practices:**
    *   **NEVER** rely solely on Jenkins' internal database for production.
    *   **ALWAYS** use encrypted channels (LDAPS, HTTPS for SAML/OAuth).
    *   **Test Failover:** Simulate LDAP/IdP outage. Ensure fallback (e.g., internal admin account) works.
    *   **Enforce Strong Passwords:** Via LDAP/IdP policy, *not* Jenkins.
    *   **Session Timeout:** Set `Security > Session Timeout` (e.g., 30 minutes). Critical for shared workstations.
    *   **Disable Guest Access:** Ensure "Allow anonymous read access" is **OFF** unless absolutely required (and then restrict jobs).

#### **2. Authorization Strategies**
*   **Matrix-Based Security:**
    *   **How it Works:** Global grid assigning permissions to Users/Groups. Permissions grouped (Administer, Job, Run, View, SCM, Agent, Credentials, etc.).
    *   **Configuration:** `Access Control > Authorization > Matrix-based security`. Add users/groups, check boxes.
    *   **Pros:** Simple for small teams. Fine-grained global control.
    *   **Cons:** **Does NOT scale.** Global scope (cannot restrict per-job). Editing matrix is error-prone. No inheritance.
    *   **Critical Limitation:** Cannot restrict permissions to *specific* jobs or folders. Only global "Job/Read" or "Job/Build".
    *   **When to Use:** Very small teams (<10 users) with minimal permission differentiation.

*   **Project-Based Matrix Authorization:**
    *   **How it Works:** Extends Matrix security. Adds a "Project-based Matrix Authorization Strategy" option. Allows defining *different* matrices **per job/folder**.
    *   **Configuration:** Requires **Matrix Authorization Strategy Plugin**. `Authorization > Project-based Matrix Authorization Strategy`. Configure global matrix *and* per-item matrices.
    *   **Pros:** Enables job/folder-level permissions. More flexible than pure global matrix.
    *   **Cons:** **Still complex to manage** at scale. Permissions defined in UI per job (not codified). Risk of inconsistent configs.
    *   **Critical Tip:** Use primarily for exceptional jobs needing unique permissions. Not ideal as primary strategy for large instances.

*   **Role-Based Strategy (Recommended for Most):**
    *   **How it Works:** Define **Roles** (Global, Project, Agent). Assign Users/Groups to Roles. Roles get Permissions.
    *   **Configuration:** Requires **Role Strategy Plugin**. `Authorization > Role-Based Strategy`.
        *   **Manage Roles:**
            *   *Global Roles:* Admin, Overall/Read, View/Create, etc. (e.g., `admin`, `dev-leads`).
            *   *Project Roles:* Patterns match job names (e.g., `frontend-.*`, `backend-.*`). Permissions like Job/Build, Job/Configure.
            *   *Slave Roles:* Patterns match agent names. Permissions like Agent/Connect, Agent/Build.
        *   **Assign Roles:** Map Users/Groups to Global/Project/Slave Roles.
    *   **Pros:** **Highly scalable.** Centralized role definition. Pattern-based job/agent assignment. Easier auditing. Supports folder permissions.
    *   **Cons:** Slightly steeper learning curve. Requires disciplined role naming.
    *   **Critical Implementation:**
        *   **Role Naming Convention:** `team-<teamname>-<role>` (e.g., `team-frontend-developer`, `team-backend-admin`).
        *   **Project Patterns:** Use regex! `^team-frontend-.*` for all frontend jobs. `^team-frontend-(prod|staging)-.*` for prod/staging.
        *   **Least Privilege:** Start with minimal permissions. Grant `Job/Read` before `Job/Build`. Never grant `Overall/Administer` broadly.
        *   **Folder Strategy:** Use **Folders Plugin**. Define roles at folder level (e.g., `folder-team-frontend`). Permissions inherit downward unless overridden.
        *   **Groups over Users:** Assign AD/LDAP groups (e.g., `jenkins-frontend-devs`) to roles, *not* individual users.
    *   **Best Practice:** **This is the gold standard for production Jenkins.** Enables separation of duties (Developers, QA, Ops, Security).

#### **3. CSRF Protection (Cross-Site Request Forgery)**
*   **What it is:** Attack where malicious site tricks user's browser into making unintended requests to Jenkins (e.g., triggering a build, deleting a job).
*   **How Jenkins Mitigates:** Requires a unique, unpredictable token (`Jenkins-Crumb`) in *every* state-changing request (POST, PUT, DELETE).
*   **Configuration:** `Security > CSRF Protection`. **ALWAYS ENABLED.**
    *   `Default Crumb Issuer`: Recommended. Uses HMAC-SHA-256.
    *   `Enable proxy compatibility`: **ONLY** if Jenkins is *always* behind a reverse proxy that sets `X-Forwarded-*` headers correctly. **Disable if unsure.**
*   **Critical Impact:**
    *   **Breaks CLI/Scripting:** Scripts using Jenkins CLI or REST API *must* fetch a crumb first:
        ```bash
        CRUMB=$(curl -s 'http://user:apiToken@jenkins-url/crumbIssuer/api/xml?xpath=//crumb' -H "Jenkins-Crumb: none")
        curl -X POST 'http://user:apiToken@jenkins-url/job/myjob/build' --data-urlencode json='{}' -H "Jenkins-Crumb:$CRUMB"
        ```
    *   **Breaks Webhooks:** Webhook senders (GitHub, GitLab) must include the crumb header. Configure webhook payload to include `Jenkins-Crumb: <crumb-value>` (fetch crumb via API first).
*   **Best Practice:** **NEVER disable CSRF protection.** Adapt scripts/webhooks to handle crumbs. Test integrations thoroughly after enabling.

#### **4. Securing Agent Communication**
*   **JNLP (Java Network Launch Protocol) Agents:**
    *   **Vulnerability:** Legacy JNLP (port 50000) uses weak encryption (or none). Traffic sniffable.
    *   **Secure JNLP (JNLP3 - Modern Standard):**
        *   **How it Works:** Uses TLS encryption for agent-controller communication. Requires Jenkins controller to have a valid TLS certificate (not self-signed for production!).
        *   **Configuration:**
            1.  Jenkins must be served **over HTTPS** (see next section). The certificate used for HTTPS is used for JNLP3.
            2.  Agent connects via `https://<jenkins-url>/tcpSlaveAgentListener/`.
            3.  Agent JVM args: `-workDir /path/to/agent/home` (avoids hardcoded paths). **NO** need for `-secret` on command line (handled securely).
        *   **Critical:** **Disable legacy JNLP port (50000)!** `Manage Jenkins > Security > Agents > TCP port for inbound agents` -> **Disable**.
    *   **Agent-to-Controller Auth:** Uses the same credentials as agent registration (public key fingerprint verification).

*   **SSH Agents:**
    *   **How it Works:** Jenkins controller connects *to* the agent via SSH (reverse of JNLP).
    *   **Security:**
        *   **Use Key-Based Auth ONLY.** **NEVER use passwords.** Generate dedicated SSH key pair (e.g., `jenkins-agent-key`).
        *   **Restrict Key:** In `~/.ssh/authorized_keys` on agent: `command="java -jar agent.jar",no-port-forwarding,no-X11-forwarding,no-agent-forwarding <public-key>`. This locks the key *only* for running the Jenkins agent.
        *   **Use Non-Privileged User:** Run agent as dedicated user (e.g., `jenkins-agent`), *not* `root`.
        *   **Firewall:** Restrict SSH access to controller's IP only.
    *   **Configuration:** `Manage Jenkins > Nodes > Configure` (for node). Set `Host`, `Credentials` (SSH key), `JavaPath`.

*   **General Agent Security Best Practices:**
    *   **Isolate Agents:** Run agents in separate network segments/VPCs from controller. Restrict controller->agent traffic to necessary ports only.
    *   **Hardened OS:** Agents should be minimal OS, patched, no unnecessary services.
    *   **Ephemeral Agents:** Prefer Docker/Kubernetes agents that are destroyed after build (reduces attack surface).
    *   **Controller Firewall:** Only allow inbound traffic from agents (for JNLP3) or *to* agents (for SSH) on specific ports. Block all other inbound.

#### **5. Securing Jenkins with HTTPS and Reverse Proxy**
*   **Why HTTPS is Non-Negotiable:**
    *   Protects credentials (logins, API tokens), CSRF tokens, build artifacts, secrets in transit.
    *   Prevents session hijacking.
    *   Required for modern browsers (Mixed Content warnings break UI).
    *   **Mandatory** for SAML/OAuth to function correctly.
*   **Implementation Options:**
    *   **Option 1: Jenkins Behind Reverse Proxy (Recommended):**
        *   **Why:** Offloads SSL/TLS, handles load balancing, provides WAF capabilities, simplifies certificate management.
        *   **Common Proxies:** Nginx, Apache HTTPD, HAProxy, Cloud Load Balancers (AWS ALB, GCP CLB).
        *   **Critical Nginx Configuration (`/etc/nginx/sites-enabled/jenkins`):**
            ```nginx
            server {
                listen 80;
                server_name jenkins.example.com;
                return 301 https://$host$request_uri; # Force HTTPS
            }

            server {
                listen 443 ssl;
                server_name jenkins.example.com;

                ssl_certificate /etc/letsencrypt/live/jenkins.example.com/fullchain.pem;
                ssl_certificate_key /etc/letsencrypt/live/jenkins.example.com/privkey.pem;
                include /etc/letsencrypt/options-ssl-nginx.conf;
                ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

                # Security Headers
                add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
                add_header X-Content-Type-Options nosniff;
                add_header X-Frame-Options "SAMEORIGIN"; # Or DENY if no embedding needed
                add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';";

                location / {
                    proxy_pass http://localhost:8080; # Jenkins HTTP port
                    proxy_set_header Host $host;
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header X-Forwarded-Proto $scheme; # **CRITICAL** for Jenkins to know it's HTTPS
                    proxy_http_version 1.1;
                    proxy_set_header Upgrade $http_upgrade;
                    proxy_set_header Connection "upgrade";
                    proxy_read_timeout 90s;
                }
            }
            ```
        *   **Critical Proxy Settings:**
            *   `X-Forwarded-Proto $scheme`: Tells Jenkins the original request was HTTPS. **Jenkins misbehaves without this!**
            *   `proxy_pass` to Jenkins **HTTP** port (8080). Do *not* configure Jenkins for HTTPS directly if behind proxy.
            *   **Jenkins Configuration:** `Manage Jenkins > Configure System > Jenkins URL` = `https://jenkins.example.com`. `Manage Jenkins > Security > Enable proxy compatibility` = **ON**.
        *   **Certificate Management:** Use Let's Encrypt (`certbot`) or enterprise PKI. **Avoid self-signed certs.**
    *   **Option 2: Jenkins Direct HTTPS (Not Recommended):**
        *   Configure `JAVA_ARGS` in `/etc/default/jenkins`: `--httpPort=-1 --httpsPort=8443 --httpsKeyStore=/var/lib/jenkins/keystore.jks --httpsKeyStorePassword=changeit`
        *   **Pros:** Simpler network config.
        *   **Cons:** Jenkins handles crypto (performance hit). Harder certificate rotation. No WAF/load balancing. **Vulnerable to slowloris attacks.** Only use if *absolutely* necessary (e.g., tiny isolated instance).

*   **Reverse Proxy Best Practices:**
    *   **Force HTTPS:** Redirect all HTTP to HTTPS (301).
    *   **HSTS:** Enable Strict-Transport-Security header (as shown above).
    *   **WAF Rules:** Implement rules to block common exploits (SQLi, XSS) at the proxy level.
    *   **Rate Limiting:** Protect against brute force (`limit_req_zone` in Nginx).
    *   **Hide Headers:** `proxy_hide_header X-Jenkins; proxy_hide_header X-Jenkins-Session;` (reduces fingerprinting).

#### **6. Audit Logging and Security Monitoring**
*   **Why:** Detect breaches, investigate incidents, meet compliance (SOC2, HIPAA, PCI-DSS), understand user activity.
*   **Core Audit Logging:**
    *   **Jenkins Audit Trail Plugin (Essential):** Logs key events to file, syslog, or database.
        *   **Critical Events to Log:**
            *   User logins/logouts (success/failure)
            *   Job creation/modification/deletion
            *   Credential creation/modification/deletion
            *   Plugin installation/update/removal
            *   Security configuration changes (Realm, Authorization)
            *   Agent connection/disconnection
            *   Sensitive operations (e.g., `Overall/Administer`)
        *   **Configuration:** `Manage Jenkins > Security > Audit Trail`. Choose output (File, Syslog, Database). Set log format (JSON preferred for parsing).
        *   **Log Location:** Default `/var/log/jenkins/audit-trail.log`. **Rotate logs!** (Use `logrotate`).
*   **Beyond Audit Trail:**
    *   **Jenkins Access Logs:** Web server access logs (Nginx/Apache) show HTTP requests (IP, URL, status code). Crucial for detecting scanning/brute force.
    *   **System Logs:** `/var/log/auth.log` (SSH logins), `/var/log/syslog` (system events).
    *   **Agent Logs:** `/var/log/jenkins-agent.log` (on agents).
*   **Security Monitoring Setup:**
    1.  **Centralized Logging:** Ship *all* logs (Jenkins audit, access, system) to SIEM (Splunk, ELK Stack, Datadog, Grafana Loki).
    2.  **Key Alerts:**
        *   Multiple failed logins from single IP (brute force).
        *   Login from unusual geographic location/new IP.
        *   `Overall/Administer` access outside business hours.
        *   Credential modification/deletion.
        *   Job deletion or SCM config change.
        *   Plugin installation from unknown source.
        *   Agent connection from unauthorized IP.
    3.  **File Integrity Monitoring (FIM):** Monitor `$JENKINS_HOME` (especially `config.xml`, `credentials.xml`, `secrets/`, `jobs/`, `plugins/`) for unexpected changes (e.g., using `aide`, `tripwire`, or cloud-native tools).
    4.  **Network Monitoring:** IDS/IPS (Suricata, Snort) watching traffic to Jenkins ports.
*   **Best Practice:** **Test your audit trail.** Simulate a security event (e.g., delete a test job) and verify it appears in logs/SIEM with sufficient detail (who, what, when, from where).

#### **7. Handling Security Advisories and Plugin Vulnerabilities**
*   **The Threat:** Jenkins core and plugins have frequent vulnerabilities (e.g., CVE-2023-45802 - RCE in Script Security Plugin). Unpatched instances are low-hanging fruit.
*   **Proactive Monitoring:**
    *   **Jenkins Security Advisories RSS Feed:** Subscribe! `https://www.jenkins.io/security/advisories/rss.xml`
    *   **National Vulnerability Database (NVD):** Search for "Jenkins" or specific plugin names.
    *   **Vulnerability Scanners:** Integrate tools like `trivy`, `snyk`, or `dependency-track` into your pipeline to scan *your* Jenkins instance/plugins (use cautiously!).
*   **Vulnerability Triage Process:**
    1.  **Identify Affected Components:** Does the advisory affect your *exact* Jenkins core version? Which plugin versions are vulnerable?
    2.  **Assess Impact:** What's the CVSS score? What's the exploit scenario? (e.g., "Requires Admin access" vs. "Remote Code Execution as Anonymous").
    3.  **Check Exposure:** Is the vulnerable component exposed? (e.g., Plugin not installed, feature disabled, behind strict firewall).
    4.  **Prioritize:** Patch Critical/High severity vulnerabilities affecting exposed components **IMMEDIATELY**. Medium/Low on schedule.
*   **Patch Management:**
    *   **Test Patches:** **NEVER patch production directly.** Have a staging/test Jenkins instance mirroring production (same plugins/versions). Test patch impact on jobs.
    *   **Rolling Updates:** For high-availability setups, update controllers one-by-one. Drain agents first.
    *   **Plugin Updates:** Update plugins *individually* where possible. Check changelogs for breaking changes. **Avoid "Update All".**
    *   **Core Updates:** Follow official upgrade guide. Backup before updating!
    *   **Emergency Patching:** For Critical RCE vulnerabilities, have a runbook: 1) Backup, 2) Patch core/plugins, 3) Restart, 4) Verify.
*   **Minimizing Risk:**
    *   **Prune Plugins:** Remove unused plugins (`Manage Jenkins > Plugin Manager > Installed`). Less code = less attack surface.
    *   **Pin Plugin Versions:** Use JCasC (Chapter 21) or `plugins.txt` to specify *exact* plugin versions. Avoid `latest`.
    *   **Use Trusted Plugins:** Prefer plugins with "Maintained" badge, active maintainers, good reviews. Avoid obscure plugins.
    *   **Sandboxing:** Ensure "Unsafe classes in sandbox" (Script Security plugin) is tightly controlled. **Disable "Approve all"** for scripts.
*   **Critical Mindset:** **Security patching is not optional.** Treat it with the same urgency as critical production outages. Schedule regular patching windows.

---

### **Chapter 21: Backup, Restore & Disaster Recovery**

#### **1. Backing Up Jenkins Home Directory**
*   **What is `$JENKINS_HOME`?** The **SINGLE MOST IMPORTANT** directory. Contains:
    *   `config.xml` (Core configuration, security settings, agent configs)
    *   `users/` (User accounts, preferences)
    *   `secrets/` (Master key, HMAC secrets, **ALL DECRYPTED CREDENTIALS!**)
    *   `credentials.xml` (Credential definitions)
    *   `jobs/` (Job configurations, `config.xml` for each job)
    *   `plugins/` (Installed plugins - `.jpi`/`.hpi` files + `*/.pinned` files)
    *   `workspace/` (Source code checkouts - **Often EXCLUDED from backup!**)
    *   `builds/` (Build history, artifacts, logs - **HUGE, often EXCLUDED!**)
    *   `fingerprints/`, `caches/`, `nodes/`, `updates/`, etc.
*   **Backup Strategy:**
    *   **Essential:** `config.xml`, `users/`, `secrets/`, `credentials.xml`, `jobs/`, `plugins/`, `nodes/` (agent configs).
    *   **Optional (Evaluate Cost/Benefit):**
        *   `builds/`: **Massive.** Only backup if build logs/artifacts are critical *and* not stored elsewhere (e.g., Artifactory, S3). Consider only last N builds.
        *   `workspace/`: **NEVER BACKUP.** Rebuildable from SCM.
    *   **Exclusion List (Typical `rsync`/`tar` exclude):**
        ```
        --exclude=workspace/** \
        --exclude=builds/**/archive/** \  # Artifacts (if stored centrally)
        --exclude=builds/**/log \          # Logs (if shipped to central logging)
        --exclude=caches/** \
        --exclude=*.log \
        --exclude=tmp/**
        ```
*   **Backup Methods:**
    *   **Filesystem Snapshot (Best):** If `$JENKINS_HOME` is on a volume supported by your infra (EBS Snapshot, Azure Disk Snapshot, ZFS snapshot). **Quiesce Jenkins first:** `sudo systemctl stop jenkins` -> Snapshot -> `sudo systemctl start jenkins`. Minimizes corruption risk.
    *   **`rsync`/`tar` (Common):**
        ```bash
        # Stop Jenkins (CRITICAL for consistency!)
        sudo systemctl stop jenkins

        # Create clean backup dir
        BACKUP_DIR="/backup/jenkins/$(date +%Y%m%d-%H%M%S)"
        mkdir -p "$BACKUP_DIR"

        # Copy essential files (adjust excludes!)
        rsync -a --delete --exclude='workspace/**' --exclude='builds/**/archive/**' \
              --exclude='builds/**/log' --exclude='caches/**' --exclude='*.log' \
              /var/lib/jenkins/ "$BACKUP_DIR/"

        # Start Jenkins
        sudo systemctl start jenkins

        # Encrypt & Ship (e.g., to S3)
        tar -czf - "$BACKUP_DIR" | openssl enc -aes-256-cbc -out "$BACKUP_DIR.tar.gz.enc" -pass pass:STRONG_PASSWORD
        aws s3 cp "$BACKUP_DIR.tar.gz.enc" s3://my-backup-bucket/
        ```
    *   **Cloud Storage:** Directly backup to S3 (AWS), Blob Storage (Azure), GCS (GCP) using `aws s3 sync`, `azcopy`, `gsutil`. **Enable versioning & MFA delete!**
*   **Critical Principles:**
    *   **STOP JENKINS** during filesystem backup. Running backup risks corrupted `config.xml`/`secrets`.
    *   **ENCRYPT** backups containing `secrets/` (which holds decrypted credentials).
    *   **TEST RESTORES** regularly (see below).
    *   **RETENTION POLICY:** Keep multiple generations (e.g., daily for 7d, weekly for 4w, monthly for 12m).

#### **2. Using ThinBackup Plugin**
*   **What it Solves:** Automated, incremental backups *without* stopping Jenkins. Backs up only changed files since last backup.
*   **How it Works:**
    *   Creates full backups periodically (e.g., daily).
    *   Creates incremental backups frequently (e.g., hourly) - only files changed since last full/incremental.
    *   Stores backups in a separate directory (e.g., `/backup/jenkins-thin`).
*   **Configuration:** `Manage Jenkins > ThinBackup`
    *   `Backup path`: `/backup/jenkins-thin` (MUST be on different filesystem than `$JENKINS_HOME`!)
    *   `Max # of daily backups`: 7
    *   `Max # of hourly backups`: 24
    *   `Schedule`: `H H/4 * * *` (Hourly) for increments, `H 2 * * *` (Daily 2AM) for fulls.
    *   `Exclude`: Set patterns to exclude `workspace`, `builds/archive`, `logs` etc. (Same as manual backup).
*   **Pros:** No Jenkins downtime. Efficient storage. Simple restore interface.
*   **Cons:**
    *   **Risk of Corruption:** If Jenkins writes to a file *while* ThinBackup is reading it, backup might be inconsistent (less likely than full fs backup while running, but possible). **Still safer than no backup!**
    *   **Not for DR:** Backups are on same server/network. **ALWAYS copy ThinBackup directory to offsite/cloud storage.**
    *   **Plugin Dependency:** Restore requires ThinBackup plugin or manual file copy.
*   **Best Practice:** Use ThinBackup for *frequent* backups, but **ALSO** take periodic full filesystem snapshots/stopped backups for true DR. **Ship ThinBackup directory offsite immediately after creation.**

#### **3. Configuration as Code (JCasC)**
*   **What it is:** Define Jenkins configuration (security, agents, tools, jobs, plugins) **in YAML files**, stored in SCM (Git). Enables versioning, code review, and reproducible setup.
*   **Core Principle:** `jenkins.yaml` replaces manual UI configuration.
*   **Key Areas Configured:**
    ```yaml
    jenkins:
      systemMessage: "Managed by JCasC. DO NOT CHANGE IN UI!"
      numExecutors: 2
      scmCheckoutRetryCount: 2
      securityRealm:
        ldap:
          servers:
            - "ldaps://ldap.example.com:636"
          rootDN: "dc=example,dc=com"
          userSearchBase: "ou=users"
          userSearch: "(sAMAccountName={0})"
          groupSearchBase: "ou=groups"
          managerDN: "cn=jenkins-svc,ou=service-accounts,dc=example,dc=com"
          managerPasswordSecret: "${LDAP_BIND_PW}" # Use credentials plugin!
      authorizationStrategy:
        roleBased:
          roles:
            global:
              - name: "admin"
                permissions:
                  - "Overall/Administer"
                assignments:
                  - "jenkins-admins" # AD Group
            projectRoles:
              - pattern: "frontend-.*"
                name: "frontend-devs"
                permissions:
                  - "Job/Build"
                  - "Job/Read"
                assignments:
                  - "frontend-team"
      nodes:
        - permanent:
            name: "linux-builder"
            remoteFS: "/home/jenkins"
            launcher:
              ssh:
                host: "builder1.internal"
                credentialsId: "ssh-key-jenkins-agent"
                jvmOptions: "-Xmx2g"
      tools:
        git: "git-2.34.1"
        jdk: "jdk17"
      plugins:
        required:
          - "git:4.10.0"
          - "workflow-api:1234.vbda1a_"
          - "role-strategy:3.0"
    credentials:
      system:
        domainCredentials:
          - credentials:
              - usernamePassword:
                  scope: GLOBAL
                  id: "artifactory-cred"
                  username: "jenkins"
                  password: "${ARTIFACTORY_TOKEN}"
                  description: "Artifactory deploy token"
    ```
*   **Implementation:**
    1.  Install **Configuration as Code Plugin**.
    2.  Create `jenkins.yaml` (and `credentials.yaml` if needed).
    3.  **Secure Secrets:** **NEVER store plaintext secrets in YAML!** Use:
        *   `credentials.yml` (encrypted with **Credentials Plugin** or **Mask Passwords Plugin**).
        *   Environment Variables (set via `docker run -e` or systemd): `password: "${ARTIFACTORY_TOKEN}"`
        *   HashiCorp Vault (via **HashiCorp Vault Plugin**): `password: "vault:secret/jenkins/artifactory#token"`
    4.  Configure JCasC: `Manage Jenkins > Configuration as Code`. Point to `jenkins.yaml` (File, URL, or SCM).
    5.  **Apply Changes:** `Reload existing configuration` (tests config) -> `Apply new configuration` (commits changes).
*   **Benefits for Backup/DR:**
    *   **Reproducible Setup:** Spin up *identical* Jenkins instance from SCM + plugins backup.
    *   **Versioned Configuration:** See who changed security settings when (`git blame jenkins.yaml`).
    *   **Disaster Recovery:** Faster recovery - deploy config + restore `JENKINS_HOME/secrets` + restore job configs (if not in JCasC).
*   **Limitations:**
    *   **Does NOT replace `JENKINS_HOME` backup:** Still need to backup `secrets/` (master.key), `jobs/` (if complex), `builds/` (if needed), `plugins/` (versions).
    *   **Job DSL vs JCasC:** JCasC configures *Jenkins itself*. Job DSL (in pipelines) configures *jobs*. Use both!
    *   **Learning Curve:** YAML syntax, understanding CasC schema.
*   **Best Practice:** **Adopt JCasC.** Start small (configure security, nodes, tools). Store `jenkins.yaml` in a **protected branch** (require PRs for changes). Use secrets management rigorously.

#### **4. Restoring Jenkins from Backup**
*   **Scenario 1: Restore on Same Server (e.g., config corruption)**
    1.  **STOP JENKINS:** `sudo systemctl stop jenkins`
    2.  **Backup Current State (Optional but Smart):** `mv /var/lib/jenkins /var/lib/jenkins.bak.$(date +%s)`
    3.  **Restore Files:**
        *   *Filesystem Backup:* `rsync -a /backup/jenkins/20231027-1000/ /var/lib/jenkins/`
        *   *ThinBackup:* Copy latest full+incremental set from `/backup/jenkins-thin/` to `/var/lib/jenkins/`
    4.  **Fix Permissions:** `chown -R jenkins:jenkins /var/lib/jenkins`
    5.  **START JENKINS:** `sudo systemctl start jenkins`
    6.  **Verify:** Check UI, logs (`/var/log/jenkins/jenkins.log`), run test job.
*   **Scenario 2: Restore to New Server (Disaster Recovery)**
    1.  **Provision New Server:** Same OS, filesystem layout as old.
    2.  **Install Jenkins:** **Exact same version** as backup was taken from (`jenkins --version`). Do *not* start Jenkins.
    3.  **Install Plugins:** Copy `plugins/*.jpi` from backup to `/var/lib/jenkins/plugins/`. **Match versions exactly.** Start Jenkins *once* to initialize, then stop.
    4.  **Restore Core Config:**
        *   Copy `config.xml`, `users/`, `secrets/`, `credentials.xml`, `nodes/` from backup to `/var/lib/jenkins/`.
        *   **CRITICAL:** `secrets/` **MUST** be restored *intact*. Mismatched `master.key` = **DECRYPTED CREDENTIALS LOST**.
    5.  **Restore Jobs (Optional):** Copy `jobs/` from backup. *Better:* Re-create jobs via JCasC/Job DSL if possible.
    6.  **Fix Permissions:** `chown -R jenkins:jenkins /var/lib/jenkins`
    7.  **Configure Reverse Proxy/HTTPS:** Match old setup.
    8.  **START JENKINS:** `sudo systemctl start jenkins`
    9.  **Verify Thoroughly:**
        *   Login as admin.
        *   Check security settings (Users, Roles, Agents).
        *   Verify credentials work (test a job using a credential).
        *   Run a representative job.
        *   Check audit logs.
*   **Critical Restoration Notes:**
    *   **Version Match:** Jenkins core & plugin versions **MUST** match the backup. Upgrading *during* restore causes failure.
    *   **Secrets are King:** If `secrets/master.key` is lost/corrupted, **ALL credentials are irrecoverable.** You *must* re-enter them manually. Backup `secrets/` religiously.
    *   **Plugin Conflicts:** If restore fails due to plugin issues, remove problematic plugin `.jpi` files from backup *before* restoring, then reinstall manually after startup.
    *   **Test Restore Process:** Document and practice this **at least quarterly**. A restore procedure only works if tested.

#### **5. Migrating Jenkins Instances**
*   **Why Migrate?** Hardware refresh, OS upgrade, cloud migration, scaling.
*   **Migration Strategies:**
    *   **A. Backup/Restore (Recommended for Major Changes):**
        *   Follow "Restore to New Server" process above.
        *   **Pros:** Clean slate, verifies backup integrity.
        *   **Cons:** Downtime during migration.
    *   **B. In-Place Upgrade (Minor Changes Only):**
        *   Upgrade OS/Jenkins version on same server.
        *   **Only** for patch-level OS/Jenkins updates. **Not** for major version jumps or cloud migration.
        *   **Critical:** Backup first! Test upgrade on staging.
    *   **C. Blue/Green Migration (Zero Downtime - Advanced):**
        1.  Build new Jenkins instance (B) alongside old (A).
        2.  Configure B identically (JCasC, restore secrets/jobs).
        3.  Point DNS/load balancer to B.
        4.  Decommission A.
        *   **Requires:** Load balancer, shared storage for `builds/` (if needed), careful credential sync.
*   **Migration Checklist:**
    1.  **Document Current State:** Jenkins version, plugin list (`jenkins-plugin-manager list`), OS, config.
    2.  **Prepare Target:** Match OS/JDK as closely as possible. Install *same* Jenkins version.
    3.  **Backup Source:** Full stopped backup (or ThinBackup snapshot).
    4.  **Restore to Target:** As per "Restore to New Server".
    5.  **Adjust Configs:**
        *   Update `JENKINS_URL` (`config.xml` or JCasC).
        *   Update agent connection details (if network changed).
        *   Update reverse proxy configs.
    6.  **Test Extensively:** All job types, credentials, security, plugins.
    7.  **Cutover:**
        *   Stop jobs on old instance.
        *   Take final backup of old instance.
        *   Restore final backup to new instance.
        *   Switch DNS/load balancer.
        *   Validate new instance.
    8.  **Decommission Old:** After successful validation period (e.g., 1 week).
*   **Critical Pitfalls:**
    *   **Plugin Incompatibility:** New OS/JDK might break native plugins (e.g., Docker, SSH Slaves). Test!
    *   **Path Changes:** `/var/lib/jenkins` vs `/jenkins/home`. Update in configs.
    *   **Firewall Rules:** New instance needs correct inbound/outbound rules.
    *   **Agent Re-registration:** Agents might need reconfiguration if controller URL changed.
    *   **Build History Loss:** If not backing up `builds/`, history is lost. Plan accordingly.

---

### **Chapter 22: Jenkins Best Practices**

#### **1. Naming Conventions**
*   **Why:** Consistency enables automation, reduces errors, improves readability.
*   **Jobs:**
    *   **Pattern:** `[team]-[project]-[env]-[purpose]` or `[project]-[feature]-[type]`
    *   **Examples:**
        *   `frontend-shoppingcart-dev-build`
        *   `backend-payment-staging-deploy`
        *   `infra-aws-terraform-apply`
        *   `myapp-featureX-unit-tests`
    *   **Rules:**
        *   Lowercase, hyphens (no spaces/underscores).
        *   Avoid `master`, `prod`, `dev` alone (use `*-prod`).
        *   Folders should reflect structure: `/teams/frontend/`, `/projects/ecommerce/`.
*   **Parameters:**
    *   Uppercase with underscores: `GIT_BRANCH`, `DEPLOY_ENV`, `ARTIFACT_VERSION`.
    *   Avoid ambiguous names: `VERSION` -> `APP_VERSION`.
*   **Credentials:**
    *   `[system]-[purpose]-[env]`: `artifactory-deploy-prod`, `github-ssh-dev`, `aws-ecr-push`.
    *   **Critical:** Scope credentials to the minimal necessary (e.g., folder-based domains).
*   **Agents/Labels:**
    *   `[os]-[arch]-[purpose]`: `linux-x64-build`, `windows-x64-qa`, `mac-arm64-ios`.
    *   Avoid generic names like `agent1`.
*   **Shared Libraries:**
    *   `jenkins-shared-lib-[team]` or `jenkins-pipeline-templates-[domain]`.
*   **Enforcement:** Use **Folder Authorization** and **Job Restrictions** plugins to enforce naming via regex.

#### **2. Pipeline Modularity**
*   **Problem:** Monolithic `Jenkinsfile` = hard to maintain, reuse, test.
*   **Solutions:**
    *   **Stages as Functions:**
        ```groovy
        def buildApp() {
            stage('Build') {
                sh 'mvn clean package'
            }
        }

        def runTests() {
            stage('Test') {
                sh 'mvn test'
            }
        }

        pipeline {
            agent any
            stages {
                buildApp()
                runTests()
            }
        }
        ```
    *   **Shared Libraries (Core Modularity):**
        *   **Structure:**
            ```
            src/
              org/
                acme/
                  PipelineUtils.groovy  // `@NonCPS` helpers
                  BuildSteps.groovy     // Reusable steps
            vars/
              buildApp.groovy           // Global pipeline step
              deployToEnv.groovy        // Global pipeline step
            resources/
              templates/                // Files for steps
            ```
        *   **Global Var Example (`vars/buildApp.groovy`):**
            ```groovy
            def call(Map config = [:]) {
                stage('Build') {
                    sh "mvn -Dversion=${config.version} clean package"
                }
            }
            ```
        *   **Usage in Jenkinsfile:**
            ```groovy
            @Library('my-shared-lib') _

            pipeline {
                agent any
                stages {
                    stage('Build') {
                        steps {
                            buildApp version: '1.0.0'
                        }
                    }
                }
            }
            ```
    *   **Composite Builds:** Use `build job: 'downstream-job'` to chain modular jobs.
*   **Benefits:** Reuse code, enforce standards, simplify `Jenkinsfile`, easier testing (test library functions).

#### **3. Immutable Infrastructure for Agents**
*   **Problem:** "Snowflake" agents (manually patched/configured) lead to "works on my agent" issues and security drift.
*   **Solution:** Treat agents like cattle, not pets. Rebuild them from a golden image for *every* build.
*   **Implementation:**
    *   **Docker Agents:**
        ```groovy
        pipeline {
            agent {
                docker {
                    image 'maven:3.8.6-jdk-11'
                    args '-v $HOME/.m2:/root/.m2' // Cache
                }
            }
            stages { ... }
        }
        ```
        *   **Best Practice:** Use trusted base images. Build custom images with dependencies pre-installed (faster builds).
    *   **Kubernetes Agents (Pod Templates):**
        ```groovy
        podTemplate {
            node {
                containerTemplate(name: 'maven', image: 'maven:3.8.6-jdk-11', command: 'cat', tty: true)
                containerTemplate(name: 'nodejs', image: 'node:18')
                ...
            }
            stages {
                stage('Build') {
                    steps {
                        container('maven') {
                            sh 'mvn clean package'
                        }
                    }
                }
            }
        }
        ```
    *   **Cloud Agents (EC2, Azure VMs):** Use pre-baked AMIs/Images with minimal OS + Jenkins agent. Scale to zero when idle.
*   **Benefits:**
    *   **Consistency:** Every build runs on a clean, identical environment.
    *   **Security:** No persistent state between builds. Vulnerabilities don't linger.
    *   **Scalability:** Spin up agents on demand.
    *   **Maintenance:** Update the image, not individual agents.

#### **4. Idempotent Builds**
*   **What:** Running the build multiple times produces the *same* result and *same* side effects (or safely handles re-runs).
*   **Why:** Essential for reliability, retrying failed steps, disaster recovery.
*   **How to Achieve:**
    *   **Source Code:** Always checkout specific commit/tag (not `master` branch).
    *   **Dependencies:** Use lock files (`package-lock.json`, `pom.xml` with versions, `requirements.txt` with hashes).
    *   **Artifacts:** Deploy to immutable storage (e.g., Artifactory with unique version, S3 with versioning). Never overwrite `latest`.
    *   **Infrastructure:** Use IaC (Terraform, CloudFormation) with state management. `terraform apply` is idempotent.
    *   **Database Migrations:** Use frameworks that track applied migrations (Flyway, Liquibase).
    *   **Avoid:** `rm -rf /tmp/*`, writing to shared mutable directories, relying on previous build state.
*   **Pipeline Example (Idempotent Deployment):**
    ```groovy
    stage('Deploy') {
        steps {
            // 1. Build artifact with unique version (from SCM tag)
            script {
                env.ARTIFACT_VERSION = sh(script: 'git describe --tags', returnStdout: true).trim()
            }
            sh 'mvn deploy -Dversion=$ARTIFACT_VERSION'

            // 2. Deploy using IaC (idempotent)
            sh 'terraform apply -var "artifact_version=$ARTIFACT_VERSION" -auto-approve'

            // 3. Register version in CMDB (idempotent PUT)
            sh 'curl -X PUT cmdb.example.com/services/myapp -d "{\"version\": \"$ARTIFACT_VERSION\"}"'
        }
    }
    ```

#### **5. Avoiding Hardcoded Values**
*   **Problem:** Hardcoded URLs, credentials, versions make pipelines inflexible and insecure.
*   **Solutions:**
    *   **Parameters:** `string(name: 'DEPLOY_ENV', defaultValue: 'staging', description: 'Environment to deploy to')`
    *   **Environment Variables:**
        *   Global: `Manage Jenkins > Configure System > Global properties > Environment variables`.
        *   Folder-level: Use **Environment Injector Plugin**.
        *   Pipeline: `environment { ARTIFACT_REPO = 'artifactory.example.com' }`
    *   **Credentials Plugin:** **ALWAYS** use `withCredentials`:
        ```groovy
        withCredentials([usernamePassword(credentialsId: 'artifactory-cred',
                                        usernameVariable: 'ARTIFACT_USER',
                                        passwordVariable: 'ARTIFACT_PASS')]) {
            sh 'curl -u $ARTIFACT_USER:$ARTIFACT_PASS artifactory.example.com/upload'
        }
        ```
    *   **Shared Libraries:** Store config in library resources or vars.
    *   **External Config:** Fetch config from Consul, Vault, or config repo at runtime (use cautiously).
*   **Critical Rule:** **NEVER commit secrets or environment-specific values to SCM.** Jenkins credentials and parameters are the *only* safe places.

#### **6. Using Parameters and Shared Libraries**
*   **Parameters (Beyond Basics):**
    *   **Active Choices / Reactive Params Plugin:** Dynamically populate params based on others (e.g., select branch -> show relevant environments).
    *   **Credentials Binding:** Parameter type `Credentials` (shows dropdown of creds).
    *   **File Parameters:** Upload files (e.g., config files) at build time.
    *   **Best Practice:** Document parameters clearly. Use defaults where safe. Restrict choices (`choice` param).
*   **Shared Libraries (Advanced Usage):**
    *   **Versioning:** Use `@Library('my-lib@v1.2')` to pin versions. Avoid `@Library('my-lib')`.
    *   **Secure Libraries:** Restrict library loading to SCM branches (JCasC: `unclassified.casc.BaseConfigurator: libraries: allowed: [main, release-*]`).
    *   **Testing Libraries:** Use Jenkins Pipeline Unit framework to test Groovy steps.
    *   **Global Variables:** Define reusable steps in `vars/` (e.g., `notifySlack.groovy`).
    *   **Classes:** Complex logic in `src/` (e.g., `GitHubHelper.groovy`).
*   **Example: Secure Parameterized Library Step (`vars/deployToEnv.groovy`):**
    ```groovy
    def call(Map config) {
        // Validate required params
        assert config.env : "env parameter is required"
        assert ['dev', 'staging', 'prod'].contains(config.env) : "Invalid env: ${config.env}"

        // Get env-specific config (from JCasC, folder props, or external source)
        def envConfig = getEnvConfig(config.env) // Helper function

        stage("Deploy to ${config.env.toUpperCase()}") {
            // Use credentials scoped to the environment
            withCredentials([string(credentialsId: "aws-${config.env}-creds", variable: 'AWS_CREDS')]) {
                sh """
                    export AWS_ACCESS_KEY_ID=\$(echo $AWS_CREDS | cut -d: -f1)
                    export AWS_SECRET_ACCESS_KEY=\$(echo $AWS_CREDS | cut -d: -f2)
                    aws s3 cp target/app.jar s3://${envConfig.bucket}/
                """
            }
        }
    }
    ```

#### **7. Monitoring and Logging**
*   **Jenkins Internal Monitoring:**
    *   **Metrics Plugin:** Exposes JVM, thread, GC, executor, queue metrics via `/metrics`. **Integrate with Prometheus!**
    *   **Monitoring Plugin:** Basic health checks (thread count, response time).
    *   **Key Metrics to Alert On:**
        *   Queue length sustained > 0
        *   Executor utilization > 80%
        *   Job duration increasing significantly
        *   Jenkins process down
        *   Disk space on `$JENKINS_HOME` < 20%
*   **Build Logging:**
    *   **Structured Logging:** Use `script { echo "[INFO] Build started" }` consistently. Avoid unstructured `sh 'echo ...'`.
    *   **Log Archiving:** Ship logs to central system (ELK, Splunk, Datadog) using **Logstash Plugin** or sidecar containers (K8s).
    *   **Annotate Logs:** Use `timestamps { ... }` and `ansiColor { ... }` in pipelines.
    *   **Log Levels:** Configure loggers (`Manage Jenkins > System Log > New Log Recorder`). Increase verbosity for debugging.
*   **Agent Monitoring:**
    *   Monitor OS metrics (CPU, Mem, Disk) on agents (Node Exporter + Prometheus).
    *   Track agent connectivity (`Manage Jenkins > Nodes > [agent] > Log`).
*   **Best Practice:** **Treat Jenkins like production app.** Monitor it as rigorously as your customer-facing services. Set up dashboards and actionable alerts.

#### **8. Documentation and Pipeline Comments**
*   **Why:** Pipelines are code. Undocumented pipelines become legacy debt.
*   **Best Practices:**
    *   **Jenkinsfile Header:**
        ```groovy
        /**
         * Project: E-Commerce Backend
         * Pipeline: CI Pipeline
         * Owner: team-backend@company.com
         * Description: Builds Java app, runs unit tests, deploys to staging.
         * Parameters:
         *   - GIT_BRANCH (string): Branch to build (default: main)
         *   - DEPLOY_ENV (choice): Environment to deploy to [dev, staging] (default: staging)
         *   - SKIP_TESTS (bool): Skip unit tests (default: false)
         * How to Trigger: SCM push, manual
         */
        ```
    *   **Stage Comments:** Briefly explain *why*, not *what*.
        ```groovy
        stage('Build') {
            // Compiles code with Maven. Requires JDK 17 (see tools config).
            steps { ... }
        }
        ```
    *   **Complex Step Comments:** Explain non-obvious logic.
        ```groovy
        sh '''
          # Workaround for Maven bug XYZ-123 (see JIRA)
          # Must run with -Dmaven.wagon.http.retryHandler.count=3
          mvn -Dmaven.wagon.http.retryHandler.count=3 clean package
        '''
        ```
    *   **External Documentation:**
        *   **Job Description Field:** Use rich text to explain purpose, ownership, links to runbooks.
        *   **Wiki Integration:** Link Jenkins jobs to Confluence/Notion docs (`[[JIRA-123]]`).
        *   **README in SCM:** Include pipeline usage/docs in project root.
    *   **Pipeline-as-Code Docs:** Document shared libraries in their repo (README.md, Javadoc-style comments).
*   **Golden Rule:** **If it's not documented, it doesn't exist.** Assume the person reading the pipeline knows nothing about the project.

---
