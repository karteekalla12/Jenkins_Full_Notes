### **Chapter 23: Jenkins with Docker**

#### **1. Running Jenkins in Docker Container**
*   **Why Docker?** Isolation, consistent environment, easy version management, simplified setup/teardown.
*   **Official Image:** `jenkins/jenkins:lts` (Linux, Java-based). `jenkins/jenkins:lts-jdk11` includes JDK 11.
*   **Key Setup Steps:**
    ```bash
    # Create persistent volume (critical for data retention!)
    docker volume create jenkins-data

    # Run Jenkins Controller (Master)
    docker run -d \
      --name jenkins-controller \
      -p 8080:8080 \
      -p 50000:50000 \
      -v jenkins-data:/var/jenkins_home \
      -v /var/run/docker.sock:/var/run/docker.sock \  # For Docker-outside-Docker (see below)
      -e JAVA_OPTS="-Djenkins.install.runSetupWizard=false" \  # Skip setup wizard for automation
      jenkins/jenkins:lts
    ```
*   **Critical Configuration:**
    *   **Volume Mounts:** `/var/jenkins_home` **MUST** be persisted (via volume or host path). Losing this = losing all jobs, configs, plugins, credentials.
    *   **Ports:** `8080` (Web UI), `50000` (Agent communication).
    *   **User Permissions:** Jenkins runs as UID `1000` inside container. Ensure host volume permissions allow this user to write (`chown -R 1000:1000 /path/to/host/volume`).
    *   **Initial Admin Password:** Found via `docker logs jenkins-controller` or in `/var/jenkins_home/secrets/initialAdminPassword` within the container/volume.
*   **Security:** Run with least privilege. Avoid `--privileged`. Use user namespaces if possible.

#### **2. Docker-in-Docker (DinD) vs. Docker-outside-of-Docker**
*   **Docker-in-Docker (DinD):**
    *   **Concept:** Run a *separate* Docker daemon *inside* the Jenkins controller container. Jenkins uses `docker` CLI inside the container to talk to this inner daemon.
    *   **Setup:**
        ```bash
        # Start DinD daemon (privileged mode required!)
        docker run -d --name dind-daemon --privileged docker:dind

        # Start Jenkins controller, linking to DinD
        docker run -d --name jenkins-controller \
          -v jenkins-data:/var/jenkins_home \
          --link dind-daemon:docker \  # Connects to DinD container
          jenkins/jenkins:lts
        ```
    *   **Pros:** Complete isolation of build Docker daemon from host. Useful for testing Docker itself.
    *   **Cons:**
        *   **Security Risk:** Requires `--privileged` (massive attack surface).
        *   **Performance Overhead:** Nested virtualization (container within container).
        *   **Complexity:** Managing two Docker daemons.
        *   **Image Pull Duplication:** Builds pull base images separately from host.
    *   **When to Use:** *Rarely.* Primarily for Jenkins plugins testing Docker functionality or specific security-isolated build requirements. **Generally discouraged for production.**

*   **Docker-outside-of-Docker (Recommended Approach):**
    *   **Concept:** Jenkins controller container *shares the host's Docker socket*. The `docker` CLI inside the container talks directly to the host's Docker daemon.
    *   **Setup:** (See "Running Jenkins" command above) `-v /var/run/docker.sock:/var/run/docker.sock`
    *   **Pros:**
        *   **Performance:** Direct access to host daemon (no nesting).
        *   **Simplicity:** No extra daemon to manage.
        *   **Image Caching:** Builds leverage host's image cache (faster pulls).
        *   **No `--privileged`:** Only needs socket access (still powerful, but less risky than full privileged).
    *   **Cons:**
        *   **Security:** Jenkins jobs *effectively have root on the host* via Docker socket access. **CRITICAL CONSIDERATION.**
        *   **Isolation:** Builds aren't isolated from each other's Docker resources on the host.
    *   **Security Mitigations (MANDATORY):**
        1.  **Dedicated Build Host:** Run Jenkins *only* on a host dedicated to CI/CD, not shared with other services.
        2.  **Agent Isolation:** **NEVER** run builds directly on the Controller. Use Docker agents (see below) or other agent types. The Controller should *only* orchestrate.
        3.  **Restrictive Policies:** Use Docker Content Trust, restrict base images, scan images.
        4.  **Socket Permissions:** Consider tools like `docker-proxy` (e.g., `socat` wrapper) to limit socket access (complex). `rootless Docker` on host is an advanced option.
    *   **When to Use:** **Overwhelmingly the standard production approach.** Security risks are manageable with proper architecture (agent isolation).

#### **3. Building Docker Images in Jenkins**
*   **Prerequisites:** Jenkins controller has Docker CLI (`docker` command available). Agent has access to Docker daemon (via socket mount for Docker agents).
*   **Pipeline Example (Declarative):**
    ```groovy
    pipeline {
        agent { dockerfile true } // Uses Docker Agent (see below) OR
        // agent { label 'docker-agent' } // If using pre-configured Docker agent label
        stages {
            stage('Build Docker Image') {
                steps {
                    script {
                        // Build image, tag it with $BUILD_ID for uniqueness
                        docker.build("my-app:${env.BUILD_ID}", "--build-arg VERSION=${env.BUILD_ID} .")
                    }
                }
            }
        }
    }
    ```
*   **Key Commands/Patterns:**
    *   `docker.build(imageName, dockerBuildArgs)`: Jenkins Pipeline Docker Plugin method.
    *   **Tags:** Always tag images meaningfully (e.g., `app:${env.GIT_COMMIT}`, `app:latest`, `app:${env.BUILD_ID}`). Avoid `latest` in production pipelines.
    *   **Build Context:** Ensure `.dockerignore` is used to exclude unnecessary files (improves speed/security).
    *   **Build Args:** Use `--build-arg` for dynamic values (e.g., version, secrets via environment variables - **use Credentials Binding!**).
    *   **Multi-stage Builds:** Highly recommended within the `Dockerfile` itself for smaller final images.
*   **Security:** Never hardcode secrets in `Dockerfile` or Jenkinsfile. Use `--build-arg` with Jenkins Credentials (bind as environment variables in the build step).

#### **4. Pushing Images to Docker Hub, ECR, or Private Registry**
*   **General Workflow:**
    1.  Authenticate to the registry.
    2.  Tag the locally built image with the *full registry path*.
    3.  Push the tagged image.
*   **Docker Hub Example:**
    ```groovy
    stage('Push to Docker Hub') {
        steps {
            script {
                docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials-id') {
                    docker.image("my-app:${env.BUILD_ID}").push("latest")
                }
            }
        }
    }
    ```
    *   `docker-hub-credentials-id`: Jenkins credential (Username/Password type) storing Docker Hub username/password.
*   **AWS ECR Example:**
    ```groovy
    stage('Push to ECR') {
        steps {
            script {
                // Get ECR login command (uses IAM role attached to EC2 instance OR Jenkins credentials)
                def ecrLogin = sh(script: 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com', returnStdout: true).trim()
                sh ecrLogin // Actually perform the login

                // Tag & Push
                sh 'docker tag my-app:${BUILD_ID} 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:${BUILD_ID}'
                sh 'docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:${BUILD_ID}'
            }
        }
    }
    ```
    *   **Authentication Methods:**
        *   **Best (EC2 Fleet):** Jenkins agent runs on EC2 instance with IAM Role granting `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:CompleteLayerUpload`, `ecr:PutImage`, `ecr:InitiateLayerUpload`. Uses `aws ecr get-login-password`.
        *   **Alternative:** Jenkins credential storing AWS Access Key ID / Secret Access Key (less secure, requires rotation).
*   **Private Registry (e.g., Harbor, Artifactory) Example:**
    ```groovy
    stage('Push to Private Registry') {
        steps {
            script {
                docker.withRegistry('https://my-registry.internal', 'private-registry-creds') {
                    docker.image("my-app:${env.BUILD_ID}").push("prod")
                }
            }
        }
    }
    ```
    *   `private-registry-creds`: Jenkins credential (Username/Password) for the private registry.
    *   **TLS:** Ensure Jenkins agents trust the private registry's CA certificate (install on host/agent image).
*   **Critical Security:** Always use TLS (HTTPS) for registries. Never push to `http://` registries in production. Scan images for vulnerabilities (Trivy, Clair) *before* pushing.

#### **5. Using Docker Agents for Builds**
*   **Why?** Avoids security risks of running builds on Controller (Docker socket access). Provides isolated, ephemeral, on-demand build environments. Ensures consistent build environments.
*   **Plugin:** **Docker Pipeline Plugin** (`docker-plugin`). *Not* the older "Docker Plugin".
*   **How it Works:**
    1.  Jenkins needs a "Docker Template" configured (Jenkins > Manage Jenkins > Configure System > Cloud > Docker).
    2.  When a job needing a Docker agent is queued, Jenkins launches a new container *from the template image*.
    3.  The Jenkins agent (JNLP) connects back to the Controller.
    4.  The build runs inside this container.
    5.  Container is destroyed after build (or idle timeout).
*   **Template Configuration (Critical Settings):**
    *   **Docker Image:** Base image for agents (e.g., `jenkins/agent:jdk11`). **Must contain:**
        *   Java Runtime (Jenkins agent requires Java)
        *   `jenkins-agent` JAR (provided by base image)
        *   Common build tools (Git, Maven/Gradle, Node.js, Docker CLI if needed - *but not Docker daemon!*)
    *   **Registry Credentials:** If image is in private registry.
    *   **Remote File System Root:** `/home/jenkins/agent` (default for `jenkins/agent` images).
    *   **Labels:** Assign a label (e.g., `docker-agent`, `maven-agent`). Jobs use `agent { label 'docker-agent' }`.
    *   **Pull Strategy:** `Pull once and update latest` (recommended).
    *   **Container Settings:**
        *   **Tty:** Checked (required for interactive shells).
        *   **Network Mode:** Often `host` (for DinD - *not recommended*) or `bridge` (default). For Docker-outside-Docker: **Mount host socket** `-v /var/run/docker.sock:/var/run/docker.sock` *as an extra host volume* in the template. **CRITICAL SECURITY NOTE:** This grants *build containers* access to the host Docker daemon. **Only do this if absolutely necessary for the build job** (e.g., building Docker images). Prefer designing builds to *not* require Docker-in-Docker. If required, use very restrictive agent labels.
*   **Pipeline Usage:**
    ```groovy
    pipeline {
        agent { label 'maven-agent' } // Uses Docker agent with this label
        stages {
            stage('Build') {
                steps {
                    sh 'mvn clean package' // Runs inside the Docker agent container
                }
            }
        }
    }
    ```
*   **Benefits:** Perfect build environment consistency, no "works on my machine" issues, automatic cleanup, easy scaling.
*   **Pitfalls:** Image pull time adds latency (mitigate with good caching/base images). Network configuration complexity. Security implications of socket mounting.

---

### **Chapter 24: Jenkins on Kubernetes**

#### **1. Deploying Jenkins on Kubernetes (Helm, YAML)**
*   **Why Kubernetes?** High availability, dynamic scaling of agents, infrastructure as code (IaC), resilience, resource efficiency.
*   **Helm (Strongly Recommended):**
    *   **Chart:** Official Jenkins Helm Chart (`https://charts.jenkins.io`)
    *   **Installation:**
        ```bash
        helm repo add jenkins https://charts.jenkins.io
        helm repo update
        helm install jenkins jenkins/jenkins --namespace jenkins --create-namespace \
          -f values.yaml  # CRITICAL: Customize values.yaml!
        ```
    *   **Key `values.yaml` Customizations:**
        ```yaml
        controller:
          # Image & Version
          image: "jenkins/jenkins"
          tag: "lts-jdk11"
          # Persistence (MANDATORY)
          persistence:
            enabled: true
            existingClaim: "jenkins-pvc"  # Pre-created PVC recommended
            # OR configure size/storageClass
          # Admin Credentials
          adminUser: "admin"
          adminPassword: "securepassword"  # BETTER: Use `existingSecret` pointing to K8s Secret
          # Service (How to access UI)
          serviceType: "ClusterIP"  # Then use Ingress, or "LoadBalancer" for direct EXTERNAL-IP
          # Ingress (Recommended for production)
          ingress:
            enabled: true
            hostName: "jenkins.mycompany.com"
            annotations:
              kubernetes.io/ingress.class: "nginx"
            tls: true
            tlsSecret: "jenkins-tls-secret"
          # Resource Requests/Limits (Essential!)
          resources:
            requests:
              memory: "2Gi"
              cpu: "500m"
            limits:
              memory: "4Gi"
              cpu: "1000m"
          # Java Options (Crucial for performance)
          javaOpts: "-Djenkins.install.runSetupWizard=false -Xmx2048m -XX:MaxRAMPercentage=50.0"
        ```
    *   **Benefits of Helm:** Easy upgrades, rollback, configuration management, dependency handling.
*   **Raw YAML (Not Recommended for Production):**
    *   Involves manually creating:
        *   `PersistentVolumeClaim` (PVC) for `/var/jenkins_home`
        *   `Deployment` for Jenkins Controller (with init containers for plugins, config)
        *   `Service` (ClusterIP + possibly LoadBalancer)
        *   `Ingress` (for external access)
        *   `Secret` (for admin password, credentials)
    *   **Why Avoid?** Extremely error-prone, misses best practices (e.g., init containers for config), hard to manage updates. Helm abstracts this complexity.

#### **2. Configuring Jenkins Controller and Dynamic Agents**
*   **Controller Configuration:**
    *   **Persistence:** PVC is non-negotiable. Use `ReadWriteOnce` storage class. Backup strategy essential (Velero, custom scripts).
    *   **Resources:** Tune `resources.requests/limits` and `javaOpts` (`-Xmx`, `-XX:MaxRAMPercentage`) based on load. Monitor memory usage!
    *   **Plugins:** Define required plugins in `values.yaml` (`controller.installPlugins`). Avoid installing via UI in K8s (ephemeral pods).
    *   **Configuration as Code (JCasC):** **ESSENTIAL.** Define controllers, security, agents, etc., via YAML injected as ConfigMap. Prevents config drift.
        ```yaml
        controller:
          JCasC:
            defaultConfig: true
            configScripts:
              welcome-message: |
                jenkins:
                  systemMessage: 'Managed by Kubernetes Helm Chart. Config in Git!'
              security: |
                jenkins:
                  authorizationStrategy:
                    loggedInUsersCanDoAnything:
                      allowAnonymousRead: false
                  securityRealm:
                    local:
                      allowsSignup: false
                      users:
                        - id: "admin"
                          password: "${ADMIN_PASSWORD}" # From K8s Secret
        ```
    *   **Backup:** Script backups of PVC to cloud storage (S3, GCS) on schedule.
*   **Dynamic Agents (The Power of K8s):**
    *   **Concept:** Agents are ephemeral Kubernetes Pods launched *on-demand* per build job, terminated after idle timeout.
    *   **Plugin:** **Kubernetes Plugin** (`kubernetes`). *Replaces* static agent configuration.
    *   **How it Works:**
        1.  Jenkins Controller runs in K8s pod.
        2.  Job requiring agent is queued.
        3.  Kubernetes Plugin requests K8s API to create a new Pod (using defined template).
        4.  Pod starts, Jenkins agent connects to Controller.
        5.  Build runs inside Pod containers.
        6.  Pod terminates after build completion + idle timeout.

#### **3. Using Kubernetes Plugin for Pod Templates**
*   **Configuration (Jenkins UI: Manage Jenkins > Configure Clouds > Kubernetes):**
    *   **Kubernetes URL:** `https://kubernetes.default.svc` (Inside cluster). **CA Certificate:** Paste cluster CA cert (from `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt` inside a pod).
    *   **Jenkins URL:** `http://jenkins-controller:8080` (Service name within K8s cluster).
    *   **Pod Templates:** Define reusable templates for different build needs (Java, Node, Python, Docker).
*   **Pod Template Structure (Example - Maven Build):**
    ```yaml
    Pod Template:
      Name: maven-agent
      Labels: maven
      Usage: Run agents only when job has matching label
      Idle Timeout: 5 (minutes - scales agents down quickly)
      Pod Retention: Keep until idle (best for cost)
      Containers:
        - Name: jnlp (MANDATORY - Jenkins agent)
          Docker Image: jenkins/inbound-agent:4.11.2-4-jdk11
          Command to run: (Leave blank - uses default)
          Arguments to pass: ${computer.jnlpmac} ${computer.name}
          Allocate TTY: true
          Resource Requests: memory=512Mi, cpu=250m
          Resource Limits: memory=1024Mi, cpu=500m
        - Name: maven
          Docker Image: maven:3.8.6-openjdk-11
          Command to run: cat (Keeps container alive)
          Arguments to pass: (None)
          Resource Requests: memory=512Mi, cpu=250m
          Resource Limits: memory=1024Mi, cpu=500m
          Always Pull Image: false
      Volumes:
        - Host Path Volume: /var/run/docker.sock -> /var/run/docker.sock (ONLY if builds need Docker - SECURITY RISK!)
        - Secret Volume: my-ssh-secret -> /home/jenkins/.ssh (For Git access)
      Node Usage: Use this node as much as possible
    ```
*   **Key Concepts:**
    *   **Pod:** The Kubernetes Pod launched for the build.
    *   **Containers:** The Pod contains multiple containers:
        *   **`jnlp` Container (Mandatory):** Runs the Jenkins agent JAR, connects to Controller. Minimal image (`jenkins/inbound-agent`).
        *   **Tool Containers (e.g., `maven`, `node`):** Run the actual build commands. Share filesystem (`/home/jenkins/agent`).
    *   **Inheritance:** Build steps run in the *first non-JNLP container* by default (`maven` in example). Can specify container per step in Pipeline.
    *   **Resource Management:** Precise CPU/Memory requests/limits per container prevent noisy neighbors.
    *   **Volumes:** Inject secrets, Docker socket (carefully!), cache directories (e.g., Maven `.m2` via PVC - use cautiously).
*   **Pipeline Usage:**
    ```groovy
    pipeline {
        agent {
            label 'maven' // Matches Pod Template label
        }
        stages {
            stage('Build') {
                steps {
                    // Runs inside 'maven' container by default
                    sh 'mvn clean package'
                }
            }
            stage('Test') {
                agent { // Override container for this stage
                    container 'node'
                }
                steps {
                    sh 'npm test' // Runs inside 'node' container
                }
            }
        }
    }
    ```

#### **4. Scaling Jenkins with K8s**
*   **Controller Scaling:**
    *   **Vertical:** Increase `resources.limits` (CPU/Memory) in Helm `values.yaml` / Deployment. Requires restart.
    *   **Horizontal (HA):** Complex. Requires shared storage (NFS, cloud storage), shared session state (Redis), careful plugin compatibility. **Not trivial.** Often better to optimize single controller or split jobs across multiple Jenkins instances. Helm chart supports it (`controller.replicaCount > 1`), but test thoroughly.
*   **Agent Scaling (The Killer Feature):**
    *   **Dynamic:** Kubernetes Plugin automatically scales agents up/down based on:
        *   Queue length (jobs waiting)
        *   `Idle Timeout` (time after build completion before Pod termination)
        *   `Pod Retention` policy
    *   **Configuration:**
        *   **Max Total # of Executors:** Global cap (K8s Cloud config).
        *   **Pod Template `Idle Timeout`:** Per-template (e.g., 5 mins for Java, 15 mins for slow builds).
        *   **Kubernetes API Rate Limits:** Tune `Container Cap` per template to avoid overwhelming K8s API.
    *   **Benefits:** Near-zero idle cost (agents only run during builds), handles massive build spikes instantly, efficient resource utilization.
    *   **Monitoring:** Track `kubernetes_pods_created`, `kubernetes_pods_terminated`, queue length. Tune `Idle Timeout` aggressively.

#### **5. CI/CD for Kubernetes Applications (Helm, Kubectl, ArgoCD)**
*   **Core Pattern:** Jenkins builds app -> Builds Docker image -> Pushes to registry -> **Deploys to K8s**.
*   **Deployment Tools Integration:**
    *   **`kubectl`:**
        ```groovy
        stage('Deploy to Dev') {
            steps {
                sh 'kubectl apply -f k8s/dev/ --namespace dev'
                sh 'kubectl rollout status deployment/my-app --namespace dev'
            }
        }
        ```
        *   **Authentication:** Use `google-gcr` plugin for GKE, `aws eks update-kubeconfig` for EKS, or ServiceAccount token mounted in Jenkins agent pod (best practice).
        *   **Security:** Agent pod needs K8s RBAC permissions (ClusterRoleBinding to a Role allowing `deployments`, `pods`, etc. in target namespace).
    *   **Helm:**
        ```groovy
        stage('Deploy with Helm') {
            steps {
                sh 'helm upgrade --install my-app ./helm-chart -f values-dev.yaml --namespace dev'
                sh 'helm test my-app --namespace dev' // Optional
            }
        }
        ```
        *   **Chart Management:** Store Helm charts in Git repo alongside app code (monorepo) or in dedicated chart repo.
        *   **Values:** Parameterize via `-f values.yaml` or `--set key=value`. Use Jenkins parameters for environment-specific values.
        *   **Registry:** If using Helm 3 with OCI registry: `helm push ./chart oci://my-registry.com/repo`
    *   **Argo CD (GitOps):**
        *   **Concept:** Jenkins builds & pushes image -> **Commits new image tag to Git (App of Apps repo)** -> Argo CD detects Git change -> Syncs cluster state.
        *   **Jenkins Pipeline Step:**
            ```groovy
            stage('Update GitOps Repo') {
                steps {
                    script {
                        // Update the image tag in the Argo CD Application manifest (e.g., values.yaml)
                        sh 'yq e \'.image.tag = "${env.BUILD_ID}"\' -i apps/my-app/values.yaml'
                        // Commit & Push to GitOps repo
                        sh '''
                            git config user.name "Jenkins"
                            git config user.email "jenkins@mycompany.com"
                            git add apps/my-app/values.yaml
                            git commit -m "Update my-app to ${BUILD_ID}"
                            git push origin main
                        '''
                    }
                }
            }
            ```
        *   **Benefits:** True GitOps, auditable deployment history, Argo CD handles sync/retry/rollback. **Jenkins only triggers the *desire* for change (Git commit), Argo CD owns the *execution*.**
        *   **Security:** Jenkins needs write access to GitOps repo (use dedicated deploy key/robot account).
*   **Best Practices:**
    *   **Immutable Tags:** Deploy using unique image tags (e.g., Git SHA), **NEVER `latest`** in production manifests.
    *   **Canary/Blue-Green:** Implement via Helm hooks, Argo Rollouts, or Flagger (integrated with Argo CD).
    *   **Approval Gates:** Use `input` step in Pipeline before production deploy.
    *   **Rollback:** Scripted rollback via Helm (`helm rollback`) or Argo CD (sync to previous Git commit).

---

### **Chapter 25: Cloud Integration**

#### **1. AWS: EC2 Fleet Plugin, S3 Artifact Storage, IAM Roles**
*   **EC2 Fleet Plugin:**
    *   **Purpose:** Launch Jenkins agents on-demand on AWS EC2 instances (On-Demand, Spot, Dedicated Hosts).
    *   **How it Works:**
        1.  Configure "Cloud" in Jenkins (Manage Jenkins > Configure Clouds > AWS EC2 Fleet).
        2.  Define an **EC2 Fleet Template**:
            *   **AMI:** Base image (e.g., Amazon Linux 2, Ubuntu). *Pre-bake common tools.*
            *   **Instance Type:** e.g., `m5.large`
            *   **Spot Fleet Request:** Enable for significant cost savings (use `maxPrice`).
            *   **Labels:** e.g., `linux-agent`, `gpu-agent`
            *   **Remote FS Root:** `/home/jenkins`
            *   **Usage:** `Utilize this node as much as possible`
            *   **Idle Termination:** 15-30 mins
            *   **IAM Instance Profile:** **CRITICAL** - Assign role granting *only* necessary permissions (S3, ECR, etc. - *not* broad admin!).
        3.  Plugin uses AWS Fleet API to launch instances matching template when builds queue.
    *   **Benefits:** Cost-effective (Spot), scales to 1000s of agents, integrates with VPC/security groups.
    *   **Security:** **IAM Role is KEY.** Principle of Least Privilege. Example minimal role policy:
        ```json
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "s3:GetObject",
                        "s3:PutObject"
                    ],
                    "Resource": "arn:aws:s3:::my-jenkins-artifacts/*"
                },
                {
                    "Effect": "Allow",
                    "Action": [
                        "ecr:GetAuthorizationToken",
                        "ecr:BatchCheckLayerAvailability",
                        "ecr:GetDownloadUrlForLayer",
                        "ecr:CompleteLayerUpload",
                        "ecr:InitiateLayerUpload",
                        "ecr:UploadLayerPart",
                        "ecr:PutImage"
                    ],
                    "Resource": "*"
                }
            ]
        }
        ```
*   **S3 Artifact Storage:**
    *   **Plugin:** **S3 Plugin** (`s3`).
    *   **Configuration:**
        *   **S3 Bucket:** `my-jenkins-artifacts`
        *   **Region:** `us-east-1`
        *   **Credentials:** **Use IAM Role** (EC2 instance profile or K8s pod identity) **NOT** Access Keys. Plugin auto-discovers credentials if running in AWS.
        *   **Path Style Access:** Usually unchecked (uses virtual-hosted style).
    *   **Usage in Pipeline:**
        ```groovy
        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                // OR (more control)
                s3Upload(
                    bucket: 'my-jenkins-artifacts',
                    path: "builds/${env.JOB_NAME}/${env.BUILD_ID}/",
                    includePathPattern: 'target/*.jar',
                    storageClass: 'STANDARD'
                )
            }
        }
        ```
    *   **Benefits:** Durable, scalable, cheaper than EBS/NAS, integrates with lifecycle policies (archive to Glacier, delete old builds).
    *   **Security:** Bucket Policy restricting access to Jenkins IAM Role. Enable S3 Encryption (SSE-S3 or SSE-KMS).

#### **2. Google Cloud: GKE, GCR, Pub/Sub Triggers**
*   **GKE (Jenkins Deployment):** Covered extensively in Chapter 24. Use Helm chart on GKE cluster. Leverage GKE-specific features:
    *   **Workload Identity:** **BEST PRACTICE** for agent authentication. Bind K8s ServiceAccount to GCP Service Account granting minimal permissions (e.g., `storage.objectAdmin` for GCS, `container.images.push` for GCR).
    *   **Node Pools:** Dedicated node pools for Jenkins controller (stable, reserved) and agents (preemptible for cost savings).
*   **GCR (Google Container Registry):**
    *   **Authentication:** **Workload Identity** (preferred) or `gcloud` auth inside agents.
    *   **Pipeline Example (Using Workload Identity):**
        ```groovy
        stage('Push to GCR') {
            steps {
                sh '''
                    # No explicit login needed if Workload Identity configured correctly!
                    docker build -t gcr.io/my-project/my-app:${BUILD_ID} .
                    docker push gcr.io/my-project/my-app:${BUILD_ID}
                '''
            }
        }
        ```
    *   **Security:** GCR IAM permissions on repository (`roles/container.imageUser` for push/pull). Enable Binary Authorization for image signing/verification.
*   **Pub/Sub Triggers:**
    *   **Purpose:** Trigger Jenkins builds based on events in GCP (e.g., Cloud Build completion, GCS upload, custom app events).
    *   **Setup:**
        1.  Create a Pub/Sub Topic (e.g., `jenkins-triggers`).
        2.  Create a Pub/Sub Subscription (Push) to Jenkins endpoint: `https://<JENKINS_URL>/google-pubsub/post`
        3.  **Enable Plugin:** **Google Cloud Pub/Sub Plugin** (`google-pubsub-messaging`).
        4.  In Jenkins Job config: **Build Triggers** > **Google Cloud Pub/Sub** > Select Topic/Subscription.
        5.  Grant Pub/Sub Subscriber role to the GCP Service Account used by the Subscription.
    *   **Example (Trigger on GCS Upload):**
        1.  Create Cloud Function triggered by GCS `google.storage.object.finalize` event.
        2.  Cloud Function publishes message to `jenkins-triggers` topic.
        3.  Jenkins job subscribed to topic triggers, processes the message (e.g., build artifact from GCS path).
    *   **Benefits:** Event-driven pipelines, decoupled architecture, integrates with wider GCP ecosystem.

#### **3. Azure: Azure VM Agents, Container Registry, AD Integration**
*   **Azure VM Agents Plugin:**
    *   **Purpose:** Launch Jenkins agents on Azure VMs (On-Demand, Spot).
    *   **Configuration:**
        *   **Cloud Name:** `azure`
        *   **Azure Credentials:** **Use Managed Identity** (strongly preferred) or Service Principal.
        *   **Virtual Machine Template:**
            *   **Labels:** `azure-agent`
            *   **Image:** Azure Marketplace image (e.g., `Canonical:0001-com-ubuntu-server-focal:20_04-lts:latest`) or Custom Image.
            *   **Size:** `Standard_DS2_v2`
            *   **Usage:** `Utilize this node as much as possible`
            *   **Idle Termination:** 15 mins
            *   **Storage Type:** `Managed`
            *   **OS Type:** `Linux`
            *   **Init Script:** Install tools (Java, Docker CLI, etc.) - **Pre-bake images is better!**
        *   **Resource Group:** Where VMs are created.
        *   **Virtual Network / Subnet:** For network isolation.
    *   **Security:** Assign **Managed Identity** to Jenkins controller (if controller in Azure) or use dedicated Service Principal with **minimal RBAC role** (e.g., `Virtual Machine Contributor` scoped to resource group).
*   **Azure Container Registry (ACR):**
    *   **Authentication:**
        *   **Best (Managed Identity):** Assign Managed Identity to Jenkins agent VM/Pool. Grant ACR role (`AcrPush`) to that identity.
        *   **Alternative:** `az acr login --name myregistry` using Service Principal credentials (less secure).
    *   **Pipeline Example (Managed Identity):**
        ```groovy
        stage('Push to ACR') {
            steps {
                sh '''
                    # Auto-login via Managed Identity (if configured correctly on agent VM)
                    docker build -t myregistry.azurecr.io/my-app:${BUILD_ID} .
                    docker push myregistry.azurecr.io/my-app:${BUILD_ID}
                '''
            }
        }
        ```
    *   **Security:** ACR firewall rules, content trust (signing), private link.
*   **Azure AD Integration:**
    *   **Plugin:** **Azure AD Plugin** (`azure-ad`).
    *   **Purpose:** Authenticate Jenkins users against Azure Active Directory (SSO).
    *   **Setup:**
        1.  Register Jenkins as an **Enterprise Application** in Azure AD.
        2.  Configure Redirect URI: `https://<JENKINS_URL>/securityRealm/finishLogin`
        3.  Grant API Permissions: `Microsoft Graph` > `User.Read` (minimum).
        4.  In Jenkins: **Security Realm** > **Azure Active Directory**.
            *   **Tenant:** `your-tenant-id.onmicrosoft.com` or `common`
            *   **Client ID:** App Registration ID
            *   **Client Secret:** Generated secret from App Registration
            *   **Group Membership:** Map Azure AD groups to Jenkins roles (e.g., `Jenkins-Admins` -> Admin).
    *   **Benefits:** Centralized user management, SSO, compliance with corporate identity policies.

#### **4. Multi-Cloud Jenkins Strategy**
*   **Why?** Avoid vendor lock-in, leverage best services per cloud, disaster recovery, cost optimization.
*   **Challenges:** Complexity, inconsistent APIs, network connectivity, security policy harmonization, skill set requirements.
*   **Strategies:**
    *   **Cloud-Agnostic Core:**
        *   Run Jenkins **Controller in a neutral location** (on-prem K8s cluster, single cloud region designated as "home").
        *   Use **Kubernetes Plugin** as the *primary* agent provisioning mechanism. Define Pod Templates compatible across clouds (e.g., standard base images).
        *   Store **Jenkins Configuration as Code (JCasC)** and **Pipeline Libraries** in Git (cloud-agnostic).
        *   Use **S3-Compatible Storage** (MinIO, Ceph) or **Artifactory** for artifacts instead of cloud-native storage.
    *   **Cloud-Specific Agents:**
        *   Configure **multiple Clouds** in Jenkins:
            *   `Kubernetes Cloud` (for GKE, EKS, AKS clusters)
            *   `AWS EC2 Fleet Cloud`
            *   `Azure VM Agents Cloud`
        *   Assign **specific labels** to agent pools (`gke-agent`, `eks-agent`, `aks-agent`, `aws-spot-agent`).
        *   Use **Pipeline Logic** to select agent based on job requirements/cost:
            ```groovy
            def chooseAgent() {
                if (env.JOB_NAME.contains('aws')) {
                    return 'aws-spot-agent'
                } else if (env.JOB_NAME.contains('azure')) {
                    return 'aks-agent'
                } else {
                    return 'gke-agent' // Default
                }
            }
            pipeline {
                agent { label chooseAgent() }
                ...
            }
            ```
    *   **Unified Identity:**
        *   Use **SAML 2.0** (Jenkins SAML Plugin) or **OpenID Connect** (Jenkins Azure AD Plugin, Google Login Plugin) federated to a central IdP (e.g., Okta, Auth0, corporate AD FS).
        *   Avoid cloud-native identity per cloud for Jenkins auth.
    *   **Infrastructure as Code (IaC):**
        *   Manage Jenkins infrastructure (Controller, agent pools) using **Terraform** or **Crossplane**. Write cloud-agnostic or cloud-specific modules.
    *   **Networking:**
        *   Establish **secure connectivity** between clouds (AWS Direct Connect / Azure ExpressRoute / GCP Interconnect, or secure tunnels like Tailscale).
        *   Use **private endpoints** (AWS PrivateLink, Azure Private Endpoint, GCP Private Service Connect) for secure registry/artifact access.
    *   **Monitoring & Logging:**
        *   Centralize logs/metrics in a single system (Datadog, Splunk, ELK stack) pulling from all cloud sources.
*   **When NOT to Go Multi-Cloud:** Most teams should **start with a single cloud**. Only adopt multi-cloud when there's a *clear, compelling business driver* (e.g., specific regulatory requirement in region X, critical application dependency on cloud Y service). Complexity cost is high.

---
