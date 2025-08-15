### **Chapter 33: End-to-End CI/CD Project (Node.js/React App with Jenkins)**

#### **1. Project Overview & Prerequisites**
*   **Tech Stack:**
    *   **Frontend:** React (Create-React-App or Vite)
    *   **Backend:** Node.js/Express.js (or similar)
    *   **SCM:** GitHub (Public/Private Repo)
    *   **CI/CD:** Jenkins (v2.4+)
    *   **Containerization:** Docker
    *   **Orchestration:** AWS ECS (Fargate preferred)
    *   **Cloud:** AWS (ECS Cluster, ALB, VPC, IAM, ECR)
    *   **Notifications:** Slack
*   **Prerequisites:**
    *   AWS Account with appropriate IAM permissions (ECS, ECR, IAM, ALB, VPC).
    *   Jenkins Master installed & secured (HTTPS, authentication).
    *   Jenkins Plugins: `GitHub Integration`, `Docker Pipeline`, `AWS Credentials`, `AWS Elastic Container Service`, `Slack Notification`, `Pipeline Utility Steps`, `Credentials Binding`, `Timestamper`.
    *   GitHub Service Account (with repo access & webhook permissions).
    *   Slack Workspace with dedicated `#ci-cd` channel & incoming webhook URL.
    *   Docker installed on Jenkins agent(s).

#### **2. Pipeline Design: Lint → Test → Build → Dockerize → Push → Deploy**
*(Declarative Pipeline Example - `Jenkinsfile`)*

```groovy
pipeline {
    agent { label 'docker-agent' } // Dedicated agent with Docker
    environment {
        APP_NAME = 'my-app'
        ECR_REPO = '123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app'
        ECS_CLUSTER = 'prod-cluster'
        ECS_SERVICE = 'my-app-service'
        ECS_TASK_DEF = 'my-app-task-def'
        SLACK_CHANNEL = '#ci-cd'
        // Use Jenkins Credentials Store! (Never hardcode)
        GITHUB_CRED = credentials('github-service-account')
        AWS_CRED = credentials('jenkins-aws-deployer')
        DOCKER_BUILDKIT = 1 // Enable BuildKit for efficiency
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm: [
                    $class: 'GitSCM',
                    branches: [[name: '*/main']], // Or feature/* for PRs
                    extensions: [[$class: 'CloneOption', depth: 1, shallow: true]],
                    userRemoteConfigs: [[
                        url: 'https://github.com/your-org/my-app-repo.git',
                        credentialsId: 'github-service-account'
                    ]]
                ]
            }
        }
        stage('Lint (Frontend & Backend)') {
            parallel {
                stage('Lint Frontend') {
                    agent { node { label 'node-agent' } }
                    steps {
                        dir('frontend') {
                            sh 'npm ci --silent' // Clean install
                            sh 'npm run lint -- --max-warnings=0' // Fail on warnings
                        }
                    }
                }
                stage('Lint Backend') {
                    agent { node { label 'node-agent' } }
                    steps {
                        dir('backend') {
                            sh 'npm ci --silent'
                            sh 'npm run lint -- --max-warnings=0'
                        }
                    }
                }
            }
        }
        stage('Test (Unit & Integration)') {
            parallel {
                stage('Test Frontend') {
                    agent { node { label 'node-agent' } }
                    steps {
                        dir('frontend') {
                            sh 'npm test -- --ci --coverage --passWithNoTests'
                            // Archive coverage report (e.g., to S3 or Jenkins)
                        }
                    }
                }
                stage('Test Backend') {
                    agent { node { label 'node-agent' } }
                    steps {
                        dir('backend') {
                            sh 'npm test -- --coverage'
                            // Archive coverage report
                        }
                    }
                }
            }
        }
        stage('Build Artifacts') {
            parallel {
                stage('Build Frontend') {
                    agent { node { label 'node-agent' } }
                    steps {
                        dir('frontend') {
                            sh 'npm ci --silent'
                            sh 'npm run build' // Outputs to /build
                            archiveArtifacts artifacts: 'build/**', fingerprint: true
                        }
                    }
                }
                stage('Build Backend') {
                    agent { node { label 'node-agent' } }
                    steps {
                        dir('backend') {
                            sh 'npm ci --silent'
                            sh 'npm run build' // Outputs to /dist
                            archiveArtifacts artifacts: 'dist/**', fingerprint: true
                        }
                    }
                }
            }
        }
        stage('Dockerize & Push') {
            steps {
                script {
                    def appVersion = readFile('VERSION').trim() // Semantic version from repo
                    def dockerfile = 'Dockerfile.prod' // Prod-specific Dockerfile
                    def imageTag = "${env.BUILD_ID}-${appVersion}" // Unique tag per build

                    // Build & Push Frontend Image
                    docker.build("${ECR_REPO}-frontend:${imageTag}", "-f ${dockerfile} --target frontend ./frontend")
                        .inside { sh 'echo "Built Frontend Image"' }
                    sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ECR_REPO}"
                    sh "docker push ${ECR_REPO}-frontend:${imageTag}"

                    // Build & Push Backend Image
                    docker.build("${ECR_REPO}-backend:${imageTag}", "-f ${dockerfile} --target backend ./backend")
                        .inside { sh 'echo "Built Backend Image"' }
                    sh "docker push ${ECR_REPO}-backend:${imageTag}"

                    // Save image tags for later deployment
                    env.FRONTEND_IMAGE = "${ECR_REPO}-frontend:${imageTag}"
                    env.BACKEND_IMAGE = "${ECR_REPO}-backend:${imageTag}"
                }
            }
        }
        stage('Deploy to AWS ECS') {
            steps {
                script {
                    // Update Task Definition with new image URIs
                    def taskDefJson = sh(
                        script: "aws ecs describe-task-definition --task-definition ${ECS_TASK_DEF} --query 'taskDefinition' --output json",
                        returnStdout: true
                    ).trim()

                    // Use jq (install on agent) to update image URIs in JSON
                    def newTaskDefJson = sh(
                        script: """
                            echo '${taskDefJson}' | jq \
                            '.containerDefinitions[] | select(.name=="frontend-container").image = "${env.FRONTEND_IMAGE}" | \
                            select(.name=="backend-container").image = "${env.BACKEND_IMAGE}"'
                        """,
                        returnStdout: true
                    ).trim()

                    // Register new revision
                    def newRevision = sh(
                        script: "echo '${newTaskDefJson}' | aws ecs register-task-definition --cli-input-json file://- --query 'taskDefinition.revision' --output text",
                        returnStdout: true
                    ).trim()

                    // Update Service to use new task definition revision
                    sh "aws ecs update-service --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE} --task-definition ${ECS_TASK_DEF}:${newRevision}"

                    // Wait for deployment stabilization (critical!)
                    sh "aws ecs wait services-stable --cluster ${ECS_CLUSTER} --services ${ECS_SERVICE}"
                }
            }
        }
    }
    post {
        success {
            slackSend channel: "${SLACK_CHANNEL}", color: 'good', message: "✅ *SUCCESS:* Build ${env.BUILD_ID} deployed to ECS! <${env.BUILD_URL}|View>"
        }
        failure {
            slackSend channel: "${SLACK_CHANNEL}", color: 'danger', message: "❌ *FAILED:* Build ${env.BUILD_ID} failed! <${env.BUILD_URL}|View>"
        }
        unstable {
            slackSend channel: "${SLACK_CHANNEL}", color: 'warning', message: "⚠️ *UNSTABLE:* Build ${env.BUILD_ID} unstable! <${env.BUILD_URL}|View>"
        }
        always {
            // Cleanup temporary files, logs, etc.
            cleanWs()
        }
    }
}
```

#### **3. Critical Implementation Details**
*   **GitHub Integration:**
    *   **Webhook:** `https://<jenkins-url>/github-webhook/` (Payload URL). Trigger: `Just the push event` (or `Pull Requests` for PR pipelines).
    *   **Authentication:** Use GitHub Personal Access Token (PAT) with `repo` scope in Jenkins Credentials (`github-service-account`).
    *   **Branch Filtering:** Use `branches: [[name: '*/main']]` for main branch deploys. For PRs, use `pullRequest` trigger and adjust stages (e.g., skip deploy).
*   **Dockerization:**
    *   **Multi-Stage Builds:** Essential! Use `--target` in `Dockerfile.prod`:
        ```dockerfile
        # Frontend Stage
        FROM node:18 as build-frontend
        WORKDIR /app
        COPY frontend/package*.json ./frontend/
        RUN npm ci --prefix frontend
        COPY frontend .
        RUN npm run build --prefix frontend

        # Backend Stage
        FROM node:18 as build-backend
        WORKDIR /app
        COPY backend/package*.json ./backend/
        RUN npm ci --prefix backend
        COPY backend .
        RUN npm run build --prefix backend

        # Final Frontend Image
        FROM nginx:alpine as frontend
        COPY --from=build-frontend /app/frontend/build /usr/share/nginx/html

        # Final Backend Image
        FROM node:18-alpine as backend
        WORKDIR /app
        COPY --from=build-backend /app/backend/dist ./dist
        COPY --from=build-backend /app/backend/package*.json ./
        RUN npm ci --only=production
        CMD ["node", "dist/index.js"]
        ```
    *   **ECR Login:** Use AWS CLI `get-login-password` (secure, no static credentials exposed). Requires Jenkins agent to have AWS credentials (via IAM Role or `AWS_CRED` env).
*   **ECS Deployment:**
    *   **Task Definition:** Must define *two* containers (`frontend-container`, `backend-container`) in the same task (sharing network namespace). Frontend served via Nginx, Backend via Node.js.
    *   **Service Configuration:** Use Rolling Update deployment type (min/max healthy %). **Crucially:** Set `Wait for service stability` (`aws ecs wait services-stable`) to prevent cascading failures.
    *   **ALB Integration:** Task Definition must expose ports (80 frontend, 3000 backend). ALB Target Groups route `/api/*` to backend, `/*` to frontend.
*   **Slack Notifications:**
    *   **Setup:** Jenkins `Manage Jenkins` > `Configure System` > `Slack` section. Enter Workspace URL, Credential (API Token with `chat:write` scope), Default Channel.
    *   **Rich Messages:** Use `slackSend` with attachments for more detail (build logs, commit info).
*   **Rollback Strategy (CRITICAL):**
    *   **Mechanism:** ECS Service Rollback via Task Definition Revision.
    *   **Trigger:**
        *   **Automated (Recommended):** Integrate with CloudWatch Alarms (e.g., 5xx errors > 1%, latency > 2s). Use AWS EventBridge rule: `Source: aws.ecs`, `Detail Type: ECS Task State Change`, `State: STOPPED`, `Exit Code: 0` (check for failures). Trigger Lambda that calls `update-service --force-new-deployment` on *previous known good revision*.
        *   **Manual:** Jenkins "Rollback" build parameter: `rollbackToRevision=123`. Pipeline stage:
            ```groovy
            stage('Rollback') {
                when { expression { params.ROLLBACK_TO_REVISION } }
                steps {
                    sh "aws ecs update-service --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE} --task-definition ${ECS_TASK_DEF}:${params.ROLLBACK_TO_REVISION} --force-new-deployment"
                    sh "aws ecs wait services-stable ..."
                }
            }
            ```
    *   **Verification:** Always include post-deploy smoke tests (e.g., `curl http://alb-dns/health` returning 200) *before* marking deploy successful. Fail pipeline if smoke test fails.

#### **4. Key Challenges & Solutions**
*   **Secrets Management:** NEVER hardcode! Use Jenkins `credentials()` binding or AWS Secrets Manager (via IAM Role on Jenkins agent/ECS tasks).
*   **Build Consistency:** Use Docker agents (`docker-agent` label) with pinned Node/Docker versions.
*   **Flaky Tests:** `--passWithNoTests` prevents failure if no tests exist (during setup). Use retry mechanisms sparingly.
*   **Image Bloat:** Multi-stage builds + `.dockerignore` (exclude `node_modules`, `*.log`).
*   **ECS Wait Timeouts:** Increase `services-stable` wait timeout if large clusters (`aws ecs wait ... --max-attempts 120`).

---

### **Chapter 34: Microservices CI/CD Pipeline**

#### **1. Core Challenges Addressed**
*   **Multiple Independent Services:** Each service has its own repo, build, test, deploy process.
*   **Shared Code/Library Dependencies:** Avoid breaking changes; manage versioning.
*   **Cross-Service Integration:** Test interactions *before* production deployment.
*   **Gradual Rollouts:** Minimize blast radius of faulty deployments.
*   **Kubernetes Complexity:** Managing manifests, deployments, rollbacks.

#### **2. Solution Architecture**
*   **Jenkins:** Orchestrator (not builder). Uses `Multibranch Pipeline` for each service repo.
*   **Shared Libraries:** Centralized Jenkins Shared Library (Groovy) for reusable pipeline steps.
*   **Kubernetes:** Cluster for staging/production (EKS, GKE, AKS).
*   **Helm:** Templating engine for Kubernetes manifests.
*   **Skaffold:** Developer inner-loop tool (build, deploy, debug) & CI integration.
*   **Service Mesh (Optional but Recommended):** Istio/Linkerd for canary routing.

#### **3. Jenkins Shared Library Implementation**
*   **Structure (`vars/` directory):**
    ```bash
    my-shared-library/
    ├── vars
    │   ├── buildDocker.groovy       # Standardized Docker build
    │   ├── deployToK8s.groovy       # Helm/Skaffold deploy
    │   ├── runIntegrationTests.groovy # Cross-service tests
    │   └── canaryDeploy.groovy      # Canary logic
    └── src
        └── com
            └── mycompany
                └── utils
                    └── Versioning.groovy # Semantic versioning logic
    ```
*   **Example: `canaryDeploy.groovy`**
    ```groovy
    def call(Map config) {
        def service = config.service
        def namespace = config.namespace
        def canaryPercent = config.canaryPercent ?: 10
        def stableRevision = config.stableRevision // Current stable version tag
        def canaryRevision = config.canaryRevision // New version tag

        stage("Canary Deploy - ${service}") {
            // 1. Deploy Canary ReplicaSet (using Helm values override or Skaffold profile)
            sh "helm upgrade ${service}-canary ./charts/${service} --namespace ${namespace} \
                --set image.tag=${canaryRevision},replicaCount=1,canary=true"

            // 2. Wait for canary pods ready
            sh "kubectl rollout status deployment/${service}-canary -n ${namespace} --timeout=5m"

            // 3. Run targeted integration tests against canary
            runIntegrationTests(service: service, namespace: namespace, target: "canary")

            // 4. Gradually shift traffic (Istio example)
            sh """
            kubectl apply -f - <<EOF
            apiVersion: networking.istio.io/v1alpha3
            kind: VirtualService
            metadata:
              name: ${service}
            spec:
              hosts:
              - ${service}
              http:
              - route:
                - destination:
                    host: ${service}
                    subset: stable
                  weight: ${100 - canaryPercent}
                - destination:
                    host: ${service}
                    subset: canary
                  weight: ${canaryPercent}
            EOF
            """

            // 5. Monitor key metrics (prometheus query) for X minutes
            def success = waitForMetrics(
                service: service,
                namespace: namespace,
                canaryPercent: canaryPercent,
                duration: "10m",
                successCriteria: [ "error_rate < 0.01", "latency_p95 < 500" ]
            )

            if (success) {
                // 6. Promote canary to stable
                sh "helm upgrade ${service} ./charts/${service} --namespace ${namespace} --set image.tag=${canaryRevision},replicaCount=3,canary=false"
                sh "kubectl delete vs ${service}-canary -n ${namespace}" // Cleanup
            } else {
                // 7. Revert traffic immediately
                sh "kubectl apply -f - <<EOF
                ... (set canary weight to 0) ...
                EOF"
                error("Canary deployment failed. Metrics degraded.")
            }
        }
    }
    ```

#### **4. Integration Testing Across Services**
*   **Strategy:**
    1.  **Contract Testing (Pact):** Service Providers publish contracts. Consumers verify against mocks *before* integration tests. Run in PR pipeline.
    2.  **Staging Environment:** Dedicated K8s namespace (`staging`) with *all* services deployed at latest commit (or specific versions).
    3.  **Test Orchestrator:** Jenkins job triggered after *all* dependent services deploy to staging:
        ```groovy
        stage('Cross-Service Integration Tests') {
            steps {
                script {
                    def services = ['auth-service', 'order-service', 'payment-service']
                    // Wait for all services to be READY in staging
                    services.each { service ->
                        sh "kubectl wait --for=condition=Available deployment/${service} -n staging --timeout=10m"
                    }
                    // Run end-to-end test suite against staging ALB
                    sh 'npm run e2e -- --baseUrl=http://staging.myapp.com'
                }
            }
        }
        ```
*   **Critical:** Use unique test data per run (e.g., `test-run-id` in headers) to avoid interference.

#### **5. Jenkins with Kubernetes (Helm & Skaffold)**
*   **Jenkins Agent on K8s:** Use `Kubernetes Plugin` to dynamically provision agents (`podTemplate`) for builds. Avoids managing static agents.
*   **Helm in CI:**
    *   **Build:** `helm dependency build` (in `charts/` dir) during Docker build stage.
    *   **Deploy:** `helm upgrade --install` as shown in `canaryDeploy.groovy`. Use `--atomic --timeout` for safety.
    *   **Versioning:** Store Helm chart versions in Chart.yaml. Use `helm package` & push to Helm repo (ChartMuseum, Artifactory) for auditability.
*   **Skaffold in CI:**
    *   **Purpose:** Streamlines build/deploy for K8s. Ideal for developer inner loop *and* CI.
    *   **CI Usage:**
        ```groovy
        stage('Deploy with Skaffold') {
            steps {
                sh "skaffold run -p ci --namespace ${env.NAMESPACE} \
                    --build-artifact=frontend:${FRONTEND_IMAGE} \
                    --build-artifact=backend:${BACKEND_IMAGE}"
            }
        }
        ```
        *   `skaffold.yaml` defines build (Docker context), deploy (Helm/Kustomize), and profiles (`ci` for optimized CI settings).
    *   **Benefits:** Automatic tagging, cache optimization, built-in rollout status checks.

#### **6. Canary Deployment Deep Dive**
*   **Phases:**
    1.  **Deploy:** New version alongside stable (reduced replicas).
    2.  **Test:** Run automated tests *specifically* against canary traffic.
    3.  **Ramp:** Gradually increase traffic (5% → 25% → 50% → 100%) over time (e.g., 15 mins/step).
    4.  **Verify:** Monitor *business* metrics (orders placed, checkout success) + technical (errors, latency).
    5.  **Promote/Abort:** Full rollout if metrics healthy; instant rollback if degradation.
*   **Traffic Shifting Mechanisms:**
    *   **Service Mesh (Istio):** Most flexible (header-based routing, gradual %). *Recommended.*
    *   **Ingress Controller (Nginx):** Limited to weight-based routing (no header rules).
    *   **Kubernetes Service:** Requires multiple Services/Deployments (less elegant).
*   **Metrics for Decision Making:**
    *   **Technical:** HTTP 5xx rate, Latency (p95, p99), Error logs, Resource usage (CPU/Mem).
    *   **Business:** Conversion rate, Checkout success rate, Key user journey success.
    *   **Source:** Prometheus (metrics), Loki/ELK (logs), Datadog/New Relic (APM).

---

### **Chapter 35: Enterprise Jenkins Setup**

#### **1. High Availability (HA) Jenkins**
*   **Why HA?** Eliminate single point of failure (SPOF). Required for 24/7 operations.
*   **Architectures:**
    *   **Active/Passive (Recommended for Simplicity):**
        *   **Master 1 (Active):** Handles all traffic.
        *   **Master 2 (Passive):** Warm standby, synchronized state.
        *   **Shared Storage:** **Critical!** All Jenkins state (`JENKINS_HOME`) on **highly available, shared storage**:
            *   **AWS:** Amazon EFS (Network File System) with Multi-AZ. *Enable encryption.*
            *   **On-Prem:** NFS cluster (e.g., GlusterFS, CephFS) or SAN with failover.
        *   **Load Balancer:** AWS ALB/NLB or HAProxy. Health checks on `/login` (HTTP 200).
        *   **Failover:** LB detects Active master failure → routes traffic to Passive. Passive becomes Active.
        *   **Pros:** Simpler, avoids split-brain. **Cons:** Passive resource idle; failover time (1-5 mins).
    *   **Active/Active (Rarely Recommended):**
        *   **Multiple Masters:** All accept traffic.
        *   **Shared Storage:** Same as Active/Passive (EFS/NFS).
        *   **Challenges:**
            *   **Split-Brain Risk:** Concurrent writes corrupt `JENKINS_HOME`.
            *   **Plugin Compatibility:** Many plugins not HA-aware (e.g., locking, cron).
            *   **Session Stickiness:** Required for UI, complicates LB config.
        *   **Only Consider If:** You have *extensive* plugin compatibility testing and custom locking mechanisms. **Not recommended for most.**
*   **Implementation Steps (Active/Passive):**
    1.  Provision 2 EC2 instances (same AMI, security group).
    2.  Mount EFS volume to `/var/lib/jenkins` on *both* instances.
    3.  Install Jenkins on both (use same version).
    4.  Configure LB (ALB) with target group pointing to both instances. Health check: `/login`, 200s, 30s interval.
    5.  **Crucial:** Set `JENKINS_OPTS="--httpPort=$HTTP_PORT --prefix=/jenkins"` in `/etc/default/jenkins` to avoid port conflicts if running locally.
    6.  Test failover: Stop Jenkins on Active master → verify Passive takes over.

#### **2. Backup and Disaster Recovery (DR) Plan**
*   **Backup Strategy (3-2-1 Rule):**
    *   **3 Copies:** Master copy + 2 backups.
    *   **2 Media Types:** EFS (primary) + S3 (backup).
    *   **1 Offsite:** S3 in different region.
*   **Backup Process:**
    1.  **Tool:** Use `thinBackup` plugin *or* custom script (more reliable).
    2.  **Script Example (`backup-jenkins.sh`):**
        ```bash
        #!/bin/bash
        set -e
        TIMESTAMP=$(date +%Y%m%d-%H%M%S)
        BACKUP_DIR="/backup/jenkins-$TIMESTAMP"
        mkdir -p $BACKUP_DIR

        # Stop Jenkins (critical for consistency!)
        systemctl stop jenkins

        # Copy JENKINS_HOME (excluding cache/workspaces)
        rsync -a --delete --exclude 'caches/' --exclude 'workspace/' \
              --exclude 'remoting/' --exclude 'fingerprints/' \
              /var/lib/jenkins/ $BACKUP_DIR/

        # Package and encrypt
        tar -czf /backup/jenkins-$TIMESTAMP.tar.gz -C /backup jenkins-$TIMESTAMP
        gpg --symmetric --cipher-algo AES256 --output /backup/jenkins-$TIMESTAMP.enc \
            /backup/jenkins-$TIMESTAMP.tar.gz

        # Upload to S3 (Cross-Region Replication enabled)
        aws s3 cp /backup/jenkins-$TIMESTAMP.enc s3://my-jenkins-backups-us-east-1/
        aws s3 cp /backup/jenkins-$TIMESTAMP.enc s3://my-jenkins-backups-eu-west-1/

        # Clean up
        rm -rf $BACKUP_DIR /backup/jenkins-$TIMESTAMP.tar.gz
        systemctl start jenkins
        ```
    3.  **Schedule:** Cron job daily at 2 AM. Retention: 30 daily, 12 weekly, 4 monthly.
*   **Disaster Recovery:**
    *   **RTO (Recovery Time Objective):** < 1 hour (HA failover). For full DR (region loss): < 4 hours.
    *   **RPO (Recovery Point Objective):** < 1 hour (frequent backups).
    *   **DR Runbook:**
        1.  Declare DR event.
        2.  Launch new Jenkins master(s) in DR region.
        3.  Restore latest backup from S3 (cross-region) to EFS volume.
        4.  Start Jenkins.
        5.  Reconfigure agents (cloud agents auto-recover; static agents need re-registration).
        6.  Validate critical pipelines.
        7.  Communicate status.

#### **3. Role-Based Access Control (RBAC) at Scale**
*   **Strategy:** Matrix-based Authorization + Project-based Matrix Authorization.
*   **Groups (LDAP/AD Integration Essential):**
    *   `jenkins-admins`: Full control (install plugins, manage nodes).
    *   `dev-lead`: Create/edit jobs in their project folder.
    *   `developer`: Read/Run builds in their project; view own credentials.
    *   `qa-engineer`: Read/Run builds; view test results.
    *   `release-manager`: Deploy to prod (explicit permission).
*   **Folder-Based Structure:**
    ```plaintext
    /
    ├── global-config (Admins only)
    ├── projects/
    │   ├── project-a/
    │   │   ├── dev (Developers, Dev-Lead)
    │   │   ├── staging (Developers, Dev-Lead, QA)
    │   │   └── prod (Release-Managers only)
    │   └── project-b/
    │       ├── dev
    │       └── prod
    └── shared-libraries (Admins, Library Maintainers)
    ```
*   **Critical Permissions:**
    *   **Credentials:** Restrict `View/Update` to specific folders/groups. Use `System Credentials` for global secrets (AWS keys).
    *   **Agents:** Restrict `Connect`/`Configure` to `jenkins-admins`.
    *   **Run:** `Build` (start), `Cancel` (stop), `Replay` (rerun stages). Restrict `Cancel` carefully.
*   **Implementation:**
    1.  Install `Role-Based Authorization Strategy` plugin.
    2.  Configure Security Realm (LDAP/AD).
    3.  Define Global Roles (Admin, Overall Read).
    4.  Define Project Roles (e.g., `project-a-dev`, `project-a-prod`).
    5.  Assign Roles to Groups/Users.
    6.  Use Folders to enforce hierarchy.

#### **4. Centralized Logging and Monitoring**
*   **Logging:**
    *   **Source:** Jenkins master logs (`/var/log/jenkins/jenkins.log`), agent logs, pipeline logs.
    *   **Shipper:** Filebeat (on Jenkins master/agents) or Fluentd.
    *   **Destination:** ELK Stack (Elasticsearch, Logstash, Kibana) or AWS CloudWatch Logs.
    *   **Key Logs to Capture:**
        *   Jenkins audit logs (`/var/log/jenkins/audit.log` - requires `audit-trail` plugin)
        *   Pipeline execution logs (structured JSON)
        *   System logs (`/var/log/syslog`, `/var/log/messages`)
    *   **Kibana Dashboard:** Track failed logins, pipeline failures, plugin errors, resource usage spikes.
*   **Monitoring:**
    *   **Metrics Source:** Jenkins `metrics` plugin + Prometheus `prometheus` plugin.
    *   **Key Metrics:**
        *   `jenkins_builds_last_duration_milliseconds` (by result)
        *   `jenkins_queue_blocked` (queue health)
        *   `jenkins_jvm_memory_used_bytes` (heap usage)
        *   `jenkins_agents_online_count`
        *   `jenkins_disk_free_megabytes` (critical for builds)
    *   **Alerting (Prometheus + Alertmanager):**
        *   `JenkinsQueueStuck{severity="critical"}`: Queue blocked > 5 mins.
        *   `JenkinsBuildFailureRate{job="prod-deploy", rate="5m"} > 0.5` (50% failure).
        *   `JenkinsDiskFree{instance="master", mount="/var"} < 10%`.
    *   **Visualization:** Grafana dashboard showing build health, queue status, resource utilization.

#### **5. Compliance and Audit Requirements**
*   **SOC 2 (Security, Availability, Confidentiality):**
    *   **Access Controls:** Enforce RBAC (as above). Regular access reviews (quarterly).
    *   **Audit Logs:** Retain Jenkins audit logs (user actions, job config changes) for 1+ year. Integrate with SIEM.
    *   **Change Management:** All pipeline/job changes via SCM (Jenkinsfiles in Git). No direct UI edits allowed. PR review required.
    *   **Vulnerability Mgmt:** Scan Jenkins plugins weekly (OWASP Dependency-Check plugin). Patch critical vulns within 7 days.
    *   **Backup Verification:** Quarterly DR test (restore from backup).
*   **GDPR (Data Privacy):**
    *   **PII Handling:** Ensure pipelines **never** log sensitive user data (emails, IDs). Use `Mask Passwords` plugin + regex masking in logs.
    *   **Right to Erasure:** Implement process to delete user data from build artifacts/logs (challenging; avoid storing PII).
    *   **Data Minimization:** Only collect build data necessary for operation. Anonymize test data.
    *   **Vendor Compliance:** Ensure cloud providers (AWS, GitHub) are GDPR-compliant (DPA signed).
*   **Audit Trail:**
    *   **Mandatory Plugins:** `audit-trail`, `config-file-provider` (track config changes).
    *   **Log Everything:** User login/logout, job creation/modification, credential changes, plugin installs, SCM commits triggering builds.
    *   **Immutable Logs:** Ship logs to WORM (Write-Once-Read-Many) storage (e.g., S3 Object Lock).
    *   **Regular Audits:** Quarterly internal audit of Jenkins access logs, RBAC assignments, and backup integrity.

---
