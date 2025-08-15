### **Chapter 28: Blue Ocean Interface**  
*The modern, visual Jenkins UI designed for Pipeline-as-Code.*

#### **1. Installing and Using Blue Ocean**  
- **What it is**: A redesigned Jenkins UI focused on Pipelines (Declarative & Scripted), replacing the classic web UI for pipeline management.  
- **Installation**:  
  ```bash
  # Via Jenkins Plugin Manager (recommended)
  jenkins-plugin-cli --plugins blueocean:1.27.8  # Always check latest version
  ```
  - **Prerequisites**: Jenkins 2.277.1+, Java 8/11, Pipeline plugin.  
  - **Post-Install**: Access via `http://<jenkins-url>/blue`. *No separate server required.*  
- **Key Workflow**:  
  1. **Create Pipeline**: Connect to GitHub/GitLab (OAuth tokens required).  
  2. **Discover Repos**: Blue Ocean scans your SCM for `Jenkinsfile`s.  
  3. **Run Pipelines**: Visual execution with real-time logs.  
- **Critical Notes**:  
  - Only manages **Pipeline projects** (not Freestyle).  
  - Requires `Jenkinsfile` in repo root (or configurable path).  
  - **Does NOT replace core Jenkins**: Classic UI still needed for system config (plugins, nodes, security).

#### **2. Visual Pipeline Editor**  
- **Purpose**: Drag-and-drop interface to build `Jenkinsfile` without YAML/DSL knowledge.  
- **How it Works**:  
  - **Stages**: Drag "Stage" box → Name it (e.g., "Build").  
  - **Steps**: Add steps inside stages (e.g., "Shell Script", "Docker Build").  
  - **Parameters**: Define `parameters` (e.g., string, choice) via UI.  
- **Output**: Generates valid Declarative Pipeline syntax in real-time (viewable in "Pipeline Script" tab).  
- **Limitations**:  
  - Only supports **Declarative Pipeline** (not Scripted).  
  - Cannot create complex logic (e.g., loops, custom functions).  
  - **Always verify generated code** – may not cover edge cases.  
- **Example Generated Snippet**:  
  ```groovy
  pipeline {
    agent any
    stages {
      stage('Build') {
        steps {
          sh 'mvn clean package'
        }
      }
    }
  }
  ```

#### **3. Personalization and UX Improvements**  
- **Key Features**:  
  - **Personalized Dashboard**: Shows *only your pipelines* (based on SCM access).  
  - **Activity Stream**: Real-time feed of pipeline runs (like GitHub timeline).  
  - **Focused Logs**: Click any step to see logs; logs auto-scroll; error lines highlighted in red.  
  - **No More "Console Output"**: Logs are structured by stage/step.  
  - **Light/Dark Mode**: User-selectable.  
- **Why it Matters**:  
  - Reduces cognitive load by 60%+ vs. classic UI (studies show faster issue diagnosis).  
  - Eliminates "where is my log?" frustration.  
  - **Critical for Teams**: New members onboard faster with visual workflow.

#### **4. Multi-branch Pipeline Visualization**  
- **How it Works**:  
  - Automatically detects branches with `Jenkinsfile`.  
  - **Branch Indexing**: Shows all branches/PRs in a single view.  
  - **Color-Coded Status**:  
    - Green: Success  
    - Red: Failed  
    - Blue: Running  
    - Grey: Not built  
- **PR Visualization**:  
  - Shows PR status directly in the branch list.  
  - Click PR → See *all* validation steps (unit tests, build, etc.).  
- **Critical Insight**:  
  - **No manual configuration** needed for branch discovery (unlike classic UI).  
  - Uses the same `Jenkinsfile` as the main branch (unless overridden via `multibranch` config).

---

### **Chapter 29: Multi-branch Pipelines**  
*Automate pipelines for every branch/PR without manual job creation.*

#### **1. Setting Up Multi-branch Pipeline Projects**  
- **Steps**:  
  1. **New Item** → Select "Multibranch Pipeline".  
  2. **Branch Sources**: Add GitHub/GitLab/Bitbucket (with credentials).  
  3. **Discover Branches**:  
     - `All Branches` (default)  
     - `Exclude PRs` / `Only PRs`  
     - Regex filter (e.g., `feature-.*`)  
  4. **Scan Triggers**:  
     - Periodic (e.g., "Scan every 1h")  
     - **Webhooks** (recommended – use GitHub "Push" + "Pull Request" events).  
- **Critical Config**:  
  - `Jenkinsfile Path`: Defaults to `Jenkinsfile` (change if using subdirs like `ci/Jenkinsfile`).  
  - **Orphaned Item Strategy**: Automatically delete old branch jobs (e.g., after 30 days).

#### **2. Automatic Branch and Pull Request Detection**  
- **How it Works**:  
  - **Branch Detection**: Scans repo → Creates pipeline job for *every branch* with `Jenkinsfile`.  
  - **PR Detection**:  
    - For GitHub: Listens to `pull_request` webhook.  
    - For GitLab: Uses "Merge Request Events".  
  - **Job Naming**: `repo-name/branch-name` (e.g., `my-app/feature-login`).  
- **Key Benefit**: Zero manual job creation for new branches.  
- **Gotcha**:  
  - PRs are built from **merge commit** (GitHub) or **head commit** (GitLab) – configure via `checkout` options.

#### **3. Per-Branch Pipeline Configuration**  
- **Override `Jenkinsfile` per branch**:  
  ```groovy
  // In Jenkinsfile (Declarative)
  pipeline {
    agent any
    stages {
      stage('Build') {
        steps {
          script {
            if (env.BRANCH_NAME == 'main') {
              sh 'mvn deploy' // Only on main
            } else {
              sh 'mvn package'
            }
          }
        }
      }
    }
  }
  ```
- **Advanced**: Use `shared libraries` to centralize branch-specific logic:  
  ```groovy
  // vars/branchConfig.groovy
  def call(String branch) {
    if (branch == 'main') return [deploy: true]
    else return [deploy: false]
  }
  ```

#### **4. PR Validation and Code Review Integration**  
- **Workflow**:  
  1. Dev pushes to branch → Opens PR.  
  2. Jenkins:  
     - Builds PR branch.  
     - Runs tests.  
     - **Posts status to SCM** (✅/❌ + link to logs).  
  3. **Merge Block**: PR cannot merge until Jenkins status is "success".  
- **Setup**:  
  - **GitHub**: Enable "Require status checks" in branch protection rules.  
  - **GitLab**: Use "Merge when pipeline succeeds".  
- **Advanced**:  
  - **Comment Triggers**: `/test` comments to rerun pipeline (via [Pipeline GitHub Plugin](https://plugins.jenkins.io/pipeline-github/)).  
  - **Quality Gates**: Fail PR if code coverage < 80% (using [Jacoco Plugin](https://plugins.jenkins.io/jacoco/)).

---

### **Chapter 30: Advanced Pipeline Techniques**  
*Beyond basic "build/test/deploy" pipelines.*

#### **1. Parallel Stages and Steps**  
- **Why**: Speed up pipelines (e.g., run tests on multiple OSes concurrently).  
- **Declarative Example**:  
  ```groovy
  stage('Parallel Tests') {
    parallel {
      stage('Linux') {
        agent { label 'linux' }
        steps { sh 'pytest' }
      }
      stage('Windows') {
        agent { label 'windows' }
        steps { bat 'pytest' }
      }
    }
  }
  ```
- **Scripted Example**:  
  ```groovy
  parallel(
    'Linux': { node('linux') { sh 'pytest' } },
    'Windows': { node('windows') { bat 'pytest' } }
  )
  ```
- **Limits**: Max 100 parallel branches (configurable via `hudson.model.ParametersDefinitionProperty.maxParallel`).

#### **2. Retry and Timeout Management**  
- **Retry Failed Stage**:  
  ```groovy
  stage('Flaky Test') {
    options { retry(3) } // Retry up to 3 times
    steps { sh 'npm test' }
  }
  ```
- **Timeout**:  
  ```groovy
  stage('Build') {
    options { timeout(time: 15, unit: 'MINUTES') } // Fail if >15m
    steps { sh 'mvn package' }
  }
  ```
- **Advanced**: Combine with `catchError`:  
  ```groovy
  stage('Deploy') {
    steps {
      catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
        timeout(time: 10, unit: 'MINUTES') {
          sh 'kubectl apply -f deploy.yaml'
        }
      }
    }
  }
  ```

#### **3. Error Handling and Recovery**  
- **Key Directives**:  
  - `catchError`: Continue pipeline after failure (sets build status).  
  - `catchException`: Only catches exceptions (not build failures).  
  - `always`: Run cleanup *regardless* of success/failure.  
- **Example**:  
  ```groovy
  stage('Deploy') {
    steps {
      catchError {
        sh 'deploy.sh'
      }
      always {
        sh 'cleanup.sh' // Runs even if deploy fails
      }
    }
  }
  ```
- **Critical Pattern**:  
  ```groovy
  stage('Recovery') {
    when { expression { currentBuild.resultIsBetterOrEqualTo('UNSTABLE') } }
    steps { sh 'rollback.sh' }
  }
  ```

#### **4. Working with Files in Workspace**  
- **Archive Artifacts**:  
  ```groovy
  post {
    success {
      archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }
  }
  ```
- **Copy Files Between Nodes**:  
  ```groovy
  stage('Transfer') {
    steps {
      stash name: 'jars', includes: 'target/*.jar'
      node('deploy-node') {
        unstash 'jars'
        sh 'scp *.jar server:/app'
      }
    }
  }
  ```
- **Clean Workspace**:  
  ```groovy
  options { 
    skipDefaultCheckout() // Disable auto-checkout
    cleanWs() // Clean workspace before build
  }
  ```

#### **5. Dynamic Job Generation**  
- **Use Case**: Create jobs for every microservice in a monorepo.  
- **Scripted Pipeline Example**:  
  ```groovy
  def services = ['auth', 'billing', 'api']
  services.each { service ->
    stage("Build ${service}") {
      steps {
        dir(service) {
          sh 'mvn package'
        }
      }
    }
  }
  ```
- **Declarative Limitation**: Requires `script {}` block:  
  ```groovy
  stages {
    script {
      def services = ['auth', 'billing']
      services.each { service ->
        stage("Build ${service}") {
          steps { sh "cd ${service} && mvn package" }
        }
      }
    }
  }
  ```

#### **6. Conditional Pipeline Execution**  
- **Branch-Based**:  
  ```groovy
  when { branch 'main' } // Only run on main
  ```
- **File Change Detection**:  
  ```groovy
  when { 
    changeset 'src/**/*.java' 
  }
  ```
- **Custom Script Condition**:  
  ```groovy
  when {
    expression { 
      return params.ENABLE_DEPLOY == 'true' 
    }
  }
  ```
- **Advanced**: Combine conditions with `allOf`/`anyOf`:  
  ```groovy
  when {
    allOf {
      branch 'main'
      expression { return env.TAG_NAME?.startsWith('v') }
    }
  }
  ```

---

### **Chapter 31: Configuration as Code (JCasC)**  
*Manage Jenkins configuration via YAML instead of UI.*

#### **1. What is JCasC?**  
- **Core Idea**: Define Jenkins controllers (security, plugins, jobs) in **version-controlled YAML**.  
- **Why it Matters**:  
  - Eliminates "works on my machine" config drift.  
  - Enables audit trails (via Git history).  
  - Critical for disaster recovery (rebuild Jenkins in minutes).  
- **Not for Pipelines**: JCasC configures *Jenkins itself*; `Jenkinsfile` still defines pipelines.

#### **2. Defining Jenkins Configuration in YAML**  
- **File Structure**: `jenkins.yaml` (typically in `/var/jenkins_home/casc_configs/`).  
- **Basic Example**:  
  ```yaml
  jenkins:
    systemMessage: "Managed by JCasC - DO NOT EDIT IN UI"
    numExecutors: 10
    scmCheckoutRetryCount: 2
  ```
- **Key Sections**:  
  - `unclassified`: Global settings (e.g., `jenkinsLocation`, `mailer`).  
  - `securityRealm`: Configure LDAP/OAuth.  
  - `authorizationStrategy`: Roles/permissions.  
- **Secrets Management**:  
  ```yaml
  unclassified:
    githubConfiguration:
      apiEndpoint: "https://api.github.com"
      manageHooks: true
      credentialsId: "${GITHUB_TOKEN}" # Injected via ENV var
  ```

#### **3. Managing Users, Roles, Jobs, and Plugins via Code**  
- **Users**:  
  ```yaml
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "${ADMIN_PASS}" # Use Jenkins credentials provider
  ```
- **Roles (via Role-Based Authorization)**:  
  ```yaml
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions: "Overall/Administer"
        project:
          - name: "dev"
            pattern: ".*-dev"
            permissions: "Job/Build, Job/Read"
  ```
- **Jobs**:  
  ```yaml
  jobs:
    - script: >
        folder('my-apps') {
          multibranchPipelineJob('web') {
            branchSources {
              git {
                id('web-repo')
                remote('https://github.com/org/web.git')
              }
            }
          }
        }
  ```
- **Plugins**:  
  ```yaml
  plugins:
    required:
      - git: 4.11.2
      - blueocean: 1.27.8
    restartJenkins: false # Default is true
  ```

#### **4. Integrating JCasC with IaC Tools**  
- **Terraform Example**:  
  ```hcl
  resource "local_file" "jenkins_casc" {
    filename = "/tmp/jenkins.yaml"
    content  = file("${path.module}/jenkins.yaml")
  }

  resource "null_resource" "jenkins" {
    provisioner "remote-exec" {
      inline = [
        "sudo mv /tmp/jenkins.yaml /var/jenkins_home/casc_configs/",
        "sudo chown jenkins:jenkins /var/jenkins_home/casc_configs/jenkins.yaml"
      ]
    }
  }
  ```
- **Ansible Example**:  
  ```yaml
  - name: Deploy JCasC config
    copy:
      src: jenkins.yaml
      dest: /var/jenkins_home/casc_configs/jenkins.yaml
      owner: jenkins
      group: jenkins
    notify: restart jenkins
  ```
- **Critical Workflow**:  
  1. Store `jenkins.yaml` in Git.  
  2. CI pipeline validates YAML (using `jcasc-validator`).  
  3. IaC tool (Terraform/Ansible) deploys to Jenkins controller.  
  4. Jenkins auto-reloads config (if `configuration-as-code` plugin is configured).

---

### **Chapter 32: Infrastructure as Code with Jenkins**  
*Jenkins as the orchestrator for IaC deployments.*

#### **1. Integrating Jenkins with Terraform, Ansible, Packer**  
- **Terraform**:  
  - **Best Practice**: Store state in remote backend (S3 + DynamoDB).  
  - **Jenkinsfile Snippet**:  
    ```groovy
    stage('Deploy') {
      steps {
        sh '''
          terraform init -backend-config="bucket=my-terraform-state"
          terraform apply -auto-approve
        '''
      }
    }
    ```
  - **Critical Security**:  
    - Use IAM roles (not credentials) via `aws sts assume-role`.  
    - Never store secrets in `Jenkinsfile` – use [Jenkins Credentials](https://www.jenkins.io/doc/book/using/using-credentials/).  
- **Ansible**:  
  - **Jenkins Setup**:  
    - Install `Ansible Plugin`.  
    - Configure SSH keys in Jenkins credentials.  
  - **Pipeline Step**:  
    ```groovy
    stage('Configure') {
      steps {
        ansiblePlaybook(
          credentialsId: 'ansible-ssh-key',
          playbook: 'deploy.yml',
          inventory: 'hosts.ini'
        )
      }
    }
    ```
- **Packer**:  
  - **Use Case**: Build golden AMIs for deployments.  
  - **Jenkinsfile**:  
    ```groovy
    stage('Build AMI') {
      steps {
        sh 'packer build -var "env=prod" packer/template.json'
      }
    }
    ```

#### **2. Automating Provisioning and Deployment**  
- **End-to-End Flow**:  
  1. **Build**: Compile code → Docker image (push to ECR).  
  2. **Provision**:  
     - Terraform creates Kubernetes cluster.  
     - Ansible configures nodes.  
  3. **Deploy**:  
     - Helm chart deploys app using new Docker image.  
- **Jenkinsfile Orchestration**:  
  ```groovy
  pipeline {
    agent any
    stages {
      stage('Build') { /* Docker build */ }
      stage('Provision') {
        parallel {
          stage('Terraform') { steps { sh 'terraform apply' } }
          stage('Packer') { steps { sh 'packer build' } }
        }
      }
      stage('Deploy') {
        steps { sh 'helm upgrade --install app ./chart' }
      }
    }
  }
  ```

#### **3. Drift Detection and Remediation**  
- **What is Drift?**: When live infrastructure diverges from IaC definitions (e.g., manual AWS console changes).  
- **Detection Tools**:  
  - **Terraform**: `terraform plan` (exit code 2 = drift detected).  
  - **AWS Config**: Tracks resource changes.  
  - **Checkov**: Static analysis of IaC files.  
- **Jenkins Automation**:  
  ```groovy
  stage('Drift Detection') {
    steps {
      script {
        def rc = sh(script: 'terraform plan -out=tfplan', returnStatus: true)
        if (rc == 2) {
          error("DRIFT DETECTED! Run 'terraform apply' to remediate")
        }
      }
    }
  }
  ```
- **Remediation Strategies**:  
  - **Auto-Apply**: Schedule nightly `terraform apply` (risky for prod).  
  - **Alert-Only**: Post drift details to Slack/email for manual review.  
  - **Policy as Code**: Use [Open Policy Agent (OPA)](https://www.openpolicyagent.org/) to block non-compliant changes.  

---
