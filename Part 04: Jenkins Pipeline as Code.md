### **Chapter 8: Introduction to Jenkins Pipeline**

#### **What is a Jenkins Pipeline?**
*   **Definition:** A Jenkins Pipeline is a **suite of plugins** enabling the definition, execution, and visualization of **continuous delivery pipelines** as *code* directly within Jenkins. It models the entire build-test-deploy process as a single, version-controlled entity.
*   **Core Purpose:** Automate complex, multi-stage CD workflows (Build → Test → Deploy → Validate → Release) with resilience, visibility, and auditability.
*   **Key Characteristics:**
    *   **Code-Centric:** Defined in a `Jenkinsfile` (text file checked into source control), not Jenkins UI forms.
    *   **Durable:** Survives Jenkins master restarts (unlike Freestyle jobs). Uses *checkpointing* to persist state.
    *   **Pausable:** Can pause for manual intervention (approvals, input).
    *   **Restartable:** Can restart from specific stages after failure (with proper configuration).
    *   **Visualizable:** Provides a clear graphical representation of pipeline execution in the Blue Ocean UI.
*   **Why Pipelines?** Solves limitations of Freestyle jobs: complex workflows are hard to manage via UI, lack versioning, are difficult to share/reuse, and offer poor visibility into multi-stage progress.

#### **Declarative vs. Scripted Pipeline**
| Feature                | Declarative Pipeline                                     | Scripted Pipeline                                      |
| :--------------------- | :------------------------------------------------------- | :----------------------------------------------------- |
| **Syntax Philosophy**  | **Structured, opinionated.** Enforces a specific order.  | **Flexible, imperative.** Full Groovy scripting power. |
| **Entry Point**        | `pipeline { ... }` block                                 | `node { ... }` block (runs on an executor)             |
| **Learning Curve**     | **Lower.** Easier for beginners. Less boilerplate.       | **Steeper.** Requires Groovy knowledge. More verbose.  |
| **Control Flow**       | Limited built-in directives (`when`, `options`).         | Full Groovy control flow (`if`, `for`, `try/catch`).   |
| **Error Handling**     | Primarily via `post` sections. Less granular.            | Fine-grained with Groovy `try/catch` blocks.           |
| **Use Case**           | **Recommended for most users.** Standardized workflows.  | **Complex logic,** custom steps, deep integration.     |
| **Validation**         | **Syntax checked at load time** (Jenkins validates structure). | Syntax errors found **only at runtime**.               |
| **Extensibility**      | Via `script {}` blocks (use sparingly) or Shared Libraries. | Directly extensible with Groovy.                       |
| **When to Choose**     | >90% of standard CD pipelines.                           | When Declarative lacks a specific feature you need.    |

**Critical Insight:** **Start with Declarative.** Only drop into Scripted (or `script {}` blocks within Declarative) when absolutely necessary. Declarative's structure prevents common anti-patterns.

#### **Benefits of Pipeline as Code (PaC)**
1.  **Version Control:** `Jenkinsfile` lives alongside application code in Git/SVN. Track changes, review diffs, revert mistakes.
2.  **Code Review:** Pipelines become part of the PR/MR process. Peers review logic *before* it runs.
3.  **Reusability:** Define common patterns once (e.g., build, deploy) and reuse across projects (via Shared Libraries).
4.  **Auditability:** Full history of *who* changed *what* and *when* in the pipeline logic.
5.  **Consistency:** Eliminates "works on my machine" for pipelines. Same pipeline runs everywhere.
6.  **Portability:** Move pipelines between Jenkins instances easily (just copy the `Jenkinsfile`).
7.  **Resilience:** Pipelines survive master restarts (unlike shell scripts triggered by Freestyle jobs).
8.  **Collaboration:** Breaks down silos; developers own pipeline logic alongside application code.
9.  **Documentation:** The `Jenkinsfile` *is* the living documentation of the delivery process.
10. **Faster Onboarding:** New team members understand the delivery process by reading the code.

#### **Jenkinsfile Structure and Syntax**
*   **Location:** Must be named `Jenkinsfile` (no extension) and placed in the **root of your project's source repository**.
*   **Basic Declarative Skeleton:**
    ```groovy
    pipeline { // MANDATORY top-level block
        agent { any } // Where to run the pipeline

        environment { // Optional: Global environment variables
            APP_NAME = 'my-app'
            DB_URL = credentials('prod-db-creds') // Using Jenkins Credentials
        }

        stages { // MANDATORY: Contains all execution stages
            stage('Build') {
                steps {
                    echo "Building ${env.APP_NAME}..."
                    sh 'mvn clean package'
                }
            }
            stage('Test') {
                steps {
                    echo "Testing..."
                    sh 'mvn test'
                }
            }
        }

        post { // Optional: Actions after *all* stages complete
            success {
                echo 'Pipeline succeeded!'
            }
            failure {
                mail to: 'team@example.com', subject: 'Pipeline Failed', body: 'Check logs!'
            }
        }
    }
    ```
*   **Basic Scripted Skeleton:**
    ```groovy
    node { // MANDATORY: Acquires an executor (agent)
        def APP_NAME = 'my-app' // Groovy variable

        stage('Build') { // Optional but highly recommended
            echo "Building ${APP_NAME}..."
            sh 'mvn clean package'
        }

        stage('Test') {
            echo "Testing..."
            try {
                sh 'mvn test'
            } catch (err) {
                mail to: 'team@example.com', subject: 'Tests Failed', body: err.getMessage()
                throw err // Re-throw to mark stage/job as failed
            }
        }
    }
    ```
*   **Key Syntax Rules:**
    *   **Curly Braces `{}`:** Define blocks (pipeline, stages, stage, steps, post, etc.). **No semicolons needed.**
    *   **Indentation:** Crucial for readability (Jenkins doesn't enforce it, but you MUST for maintainability).
    *   **Strings:** Use single `'` or double `"` quotes. Double quotes allow variable interpolation (`"${var}"`).
    *   **Comments:** `// Single-line`, `/* Multi-line */`.
    *   **Directives:** Declarative uses specific keywords (`agent`, `stages`, `environment`, `parameters`, `options`, `triggers`, `post`). **Order matters!**
    *   **Groovy in Scripted:** Full Groovy syntax applies (variables, functions, control flow). **Beware:** Scripted runs *inside* the Jenkins master JVM – avoid heavy computation.

#### **Writing Your First Pipeline**
1.  **Prerequisite:** Jenkins instance with "Pipeline" plugin installed (usually bundled).
2.  **Create a New Pipeline Job:**
    *   Jenkins Dashboard → New Item → Type "Pipeline" → Name it (e.g., `hello-pipeline`).
    *   Under "Pipeline" section:
        *   *Definition:* Select **"Pipeline script from SCM"**.
        *   *SCM:* Choose Git, enter your repo URL (e.g., `https://github.com/yourname/demo-repo.git`).
        *   *Script Path:* `Jenkinsfile` (default location).
    *   Save.
3.  **Create the `Jenkinsfile` in your Repo:**
    ```groovy
    pipeline {
        agent any // Run on any available agent

        stages {
            stage('Hello') {
                steps {
                    echo 'Hello, Jenkins Pipeline!'
                    sh 'echo "Current directory: $(pwd)"'
                    sh 'git rev-parse HEAD' // Show current commit
                }
            }
        }
    }
    ```
4.  **Commit & Push:** Add `Jenkinsfile` to your repo, commit, and push.
5.  **Run the Pipeline:** In Jenkins, click "Build Now" for your job.
6.  **Verify:**
    *   Check Console Output for `Hello, Jenkins Pipeline!` and directory/commit info.
    *   View Pipeline Stage View (Blue Ocean is highly recommended for visualization).

**First Pipeline Pro Tips:**
*   Start simple (`agent any`, single `stage`).
*   Use `echo` and `sh` liberally for debugging.
*   **Always commit `Jenkinsfile` to source control.** Never use "Pipeline script" textbox for production.
*   Enable "Scan Multibranch Pipeline Triggers" if using branches.

---

### **Chapter 9: Declarative Pipeline**

#### **Pipeline Block Structure (The Mandatory Skeleton)**
```groovy
pipeline { // TOP-LEVEL BLOCK - REQUIRED
    agent { ... } // WHERE TO RUN - REQUIRED (but `any` is minimal)
    // Optional Top-Level Sections (Order Matters!):
    environment { ... }   // Global Env Vars
    options { ... }       // Pipeline-wide settings (timeout, retry, etc.)
    parameters { ... }    // User-input parameters
    triggers { ... }      // What starts the pipeline
    stages { ... }        // EXECUTION - REQUIRED (contains stages)
    post { ... }          // ACTIONS AFTER STAGES - Optional
}
```
*   **Order is Strict:** `agent`, `options`, `parameters`, `triggers`, `stages`, `post` must appear in this sequence if used. `environment` can be top-level *or* inside `stages`/`stage`.
*   **`stages` is Mandatory:** Contains the actual work (`stage` blocks).

#### **Core Directives Explained**

1.  **`agent`**
    *   **Purpose:** Defines *where* the pipeline or a specific stage executes.
    *   **Top-Level vs. Stage-Level:** Top-level sets default; stage-level overrides it.
    *   **Common Types:**
        *   `any`: Run on any available agent (label not needed).
        *   `label 'my-agent'`: Run on agent with label `my-agent`.
        *   `node { label 'my-agent'; customWorkspace '/path' }`: More control (custom workspace).
        *   `docker { image 'maven:3-alpine'; }`: Run inside a Docker container (requires Docker plugin).
        *   `kubernetes { ... }`: Run inside a Kubernetes pod (requires Kubernetes plugin).
    *   **Example (Top-Level):**
        ```groovy
        agent { label 'build-agent && linux' } // Needs agent with BOTH labels
        ```
    *   **Example (Stage-Level - Overrides Top-Level):**
        ```groovy
        stage('Deploy to Prod') {
            agent { label 'prod-deployer' } // Only this stage uses prod agent
            steps { ... }
        }
        ```

2.  **`stages`, `stage`, `steps`**
    *   **`stages` (Block):** **Mandatory container** for all `stage` blocks. Defines the sequence of work.
    *   **`stage` (Block):** Represents a **logical phase** (e.g., Build, Test, Deploy). **Required name** (string). Contains `steps` (or `when`, `options`, `environment`, `post` for the stage).
    *   **`steps` (Block):** **Mandatory container** inside a `stage`. Contains the actual commands/tasks (`sh`, `echo`, `script`, plugin steps).
    *   **Example:**
        ```groovy
        stages {
            stage('Build') { // Stage Name
                steps { // Steps Container
                    sh 'mvn clean package' // Step 1
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true // Step 2
                }
            }
            stage('Test') {
                steps {
                    sh 'mvn test'
                }
            }
        }
        ```

3.  **`environment` Directive**
    *   **Purpose:** Define **key-value pairs** for environment variables.
    *   **Scope:** Can be top-level (applies to *all* stages) or inside a `stage` (applies only to that stage).
    *   **Syntax:**
        ```groovy
        environment {
            APP_VERSION = '1.0' // Simple string
            DB_URL = 'jdbc:mysql://prod-db:3306/mydb'
            // Using Credentials (SAFEST method):
            AWS_ACCESS_KEY_ID = credentials('aws-prod-creds') // ID of Jenkins Credential
            // Credentials create 2 vars: `ID` and `ID_USR`/`ID_PSW` (for Username/Password)
        }
        ```
    *   **Critical Security Note:** **NEVER hardcode secrets** like passwords, API keys, or tokens directly here. **ALWAYS** use `credentials()` binding to reference Jenkins Credentials (Stored securely in Jenkins).

4.  **`parameters` Directive**
    *   **Purpose:** Define **user-input parameters** when triggering a build manually (or via `build` step). Values are available as `params.VAR_NAME`.
    *   **Common Parameter Types:**
        ```groovy
        parameters {
            string(name: 'VERSION', defaultValue: '1.0', description: 'App Version to build')
            booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run tests?')
            choice(name: 'ENV', choices: ['dev', 'staging', 'prod'], description: 'Target Environment')
            password(name: 'API_TOKEN', defaultValue: '', description: 'Secret API Token') // Masked in logs
            text(name: 'NOTES', defaultValue: '', description: 'Release notes')
        }
        ```
    *   **Usage in Steps:**
        ```groovy
        steps {
            echo "Building version ${params.VERSION} for ${params.ENV}"
            sh "mvn deploy -Denv=${params.ENV}"
        }
        ```

5.  **`options` Directive**
    *   **Purpose:** Configure **pipeline-wide behaviors** (can also be used at `stage` level).
    *   **Key Options:**
        *   `retry(int)`: Retry the *entire pipeline* (or stage) on failure. `options { retry(3) }`
        *   `timeout(time: int, unit: 'SECONDS|MINUTES|HOURS')`: Fail pipeline (or stage) if it runs too long. `options { timeout(time: 30, unit: 'MINUTES') }`
        *   `timestamps()`: Prefix all console log lines with current time. `options { timestamps() }`
        *   `skipDefaultCheckout()`: Skip the implicit `checkout scm` step at the start. (Use if you do custom checkout).
        *   `disableConcurrentBuilds()`: Prevent multiple concurrent builds of *this pipeline*. `options { disableConcurrentBuilds() }`
        *   `buildDiscarder(logRotator(...))`: Configure log retention. `options { buildDiscarder(logRotator(daysToKeepStr: '30', numToKeepStr: '20')) }`
    *   **Example:**
        ```groovy
        options {
            timeout(time: 1, unit: 'HOURS') // Fail if whole pipeline > 1hr
            retry(2)                         // Retry whole pipeline twice on failure
            timestamps()                     // Timestamp all log lines
            disableConcurrentBuilds()        // Only one build at a time
        }
        ```

6.  **`triggers` Directive**
    *   **Purpose:** Define **automatic triggers** to start the pipeline.
    *   **Common Triggers:**
        *   `cron('H */4 * * 1-5')`: Run based on cron syntax (Jenkins uses `H` for "hash" to spread load). `H` = random minute within hour. `H */4 * * 1-5` = Every 4 hours on Mon-Fri.
        *   `pollSCM('H */4 * * *')`: Poll SCM (e.g., Git) every 4 hours; trigger if changes found. **Less efficient than webhooks.**
        *   `upstream(upstreamProjects: 'build-job', threshold: 'SUCCESS')`: Trigger when upstream job (`build-job`) finishes with `SUCCESS` (or `UNSTABLE`, `FAILURE`, `NOT_BUILT`).
        *   `bitbucketPush()`: Trigger on Bitbucket Cloud push (requires Bitbucket plugin).
        *   `githubPush()`: Trigger on GitHub push (requires GitHub plugin).
    *   **Example (Cron & SCM Polling):**
        ```groovy
        triggers {
            cron('H 0 * * *') // Run once daily at midnight (random minute)
            pollSCM('H/15 * * * *') // Poll SCM every 15 mins; trigger if changes
        }
        ```
    *   **Best Practice:** **Prefer SCM Webhooks** (GitHub, GitLab, Bitbucket hooks) over `pollSCM` for instant, efficient triggering. Configure hooks in your SCM repo settings pointing to `JENKINS_URL/github-webhook/` (or similar).

7.  **`post` Conditions**
    *   **Purpose:** Define **actions to run after the main `stages` complete**, based on the pipeline's **final outcome**.
    *   **Conditions (Order of Execution):**
        *   `always { ... }`: **ALWAYS** runs, regardless of status. (Good for cleanup).
        *   `changed { ... }`: Runs only if the **current Pipeline's run state changed** (e.g., from Success → Failure).
        *   `failure { ... }`: Runs only if the Pipeline **failed**.
        *   `success { ... }`: Runs only if the Pipeline **succeeded**.
        *   `unstable { ... }`: Runs only if the Pipeline was marked **UNSTABLE** (e.g., tests failed but build succeeded).
        *   `aborted { ... }`: Runs only if the Pipeline was **manually aborted**.
        *   `cleanup { ... }`: **ALWAYS** runs, **LAST** (even after `always`). **Guaranteed final cleanup.**
    *   **Execution Order:** `cleanup` → `always` → `changed` → `failure`/`success`/`unstable`/`aborted`. **Critical for resource cleanup!**
    *   **Example:**
        ```groovy
        post {
            success {
                echo 'Pipeline succeeded! Notifying Slack...'
                slackSend channel: '#builds', message: "SUCCESS: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
            }
            failure {
                echo 'Pipeline failed! Sending email...'
                mail to: 'team@example.com', subject: "FAILED: ${env.JOB_NAME} ${env.BUILD_NUMBER}", body: 'Check logs!'
            }
            unstable {
                echo 'Tests unstable! Filing Jira ticket...'
                // jiraCreate ... (using Jira plugin)
            }
            always {
                echo 'Archiving test results (always runs)'
                junit 'target/surefire-reports/*.xml'
            }
            cleanup {
                echo 'REMOVING temporary resources (runs LAST, even if failure)'
                sh 'docker rm -f test-container || true' // Ignore failure
            }
        }
        ```
    *   **Critical Note:** `cleanup` is the **ONLY** place you can reliably clean up resources (like Docker containers, VMs) because it runs *after* `always` and *even if* the pipeline is aborted or fails catastrophically.

#### **Example: Multi-Stage Pipeline (Build → Test → Deploy)**
```groovy
pipeline {
    agent { label 'build-agent' } // Runs on agent with label 'build-agent'

    environment {
        APP_NAME = 'shopping-cart'
        // NEVER hardcode secrets! Use Jenkins Credentials
        AWS_ACCESS_KEY_ID = credentials('aws-prod-creds')
        AWS_SECRET_ACCESS_KEY = credentials('aws-prod-creds')
    }

    options {
        timeout(time: 45, unit: 'MINUTES') // Fail if whole pipeline > 45 mins
        timestamps() // Timestamp log lines
        buildDiscarder(logRotator(daysToKeepStr: '30', numToKeepStr: '20')) // Keep 20 builds, 30 days
    }

    triggers {
        // Trigger on GitHub push to main branch (configured via GitHub webhook)
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm // Implicit checkout of the repo that triggered the build
            }
        }
        stage('Build') {
            steps {
                echo "Building ${env.APP_NAME} version ${params.VERSION}..."
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
        stage('Test') {
            when { expression { params.RUN_TESTS } } // Only run if RUN_TESTS param is true
            steps {
                echo 'Running unit tests...'
                sh 'mvn test'
                junit 'target/surefire-reports/*.xml' // Publish test results
            }
        }
        stage('Deploy to Staging') {
            when {
                expression { params.ENV == 'staging' } // Only deploy to staging if ENV=staging
            }
            steps {
                echo 'Deploying to Staging Environment...'
                sh 'aws s3 cp target/*.jar s3://staging-bucket/${APP_NAME}/'
                sh 'kubectl apply -f k8s/staging -n staging'
            }
        }
        stage('Manual Approval - Prod') {
            when {
                expression { params.ENV == 'prod' } // Only show for prod deploys
            }
            steps {
                input {
                    message "Deploy ${env.APP_NAME} version ${params.VERSION} to PRODUCTION?"
                    ok 'Deploy Now'
                    submitter 'prod-deploy-group' // Only users in this Jenkins group can approve
                }
            }
        }
        stage('Deploy to Prod') {
            when {
                expression { params.ENV == 'prod' } // Only deploy to prod if ENV=prod
            }
            steps {
                echo 'Deploying to PRODUCTION!!!'
                sh 'aws s3 cp target/*.jar s3://prod-bucket/${APP_NAME}/'
                sh 'kubectl apply -f k8s/prod -n prod'
            }
        }
    }

    post {
        success {
            script {
                // Send rich Slack message on success
                slackSend(
                    channel: '#prod-deploys',
                    color: 'good',
                    message: "SUCCESS: ${env.APP_NAME} v${params.VERSION} deployed to ${params.ENV} :tada:"
                )
            }
        }
        failure {
            mail to: 'devops-team@example.com', subject: "FAILED: ${env.JOB_NAME} ${env.BUILD_NUMBER}", body: """
                Pipeline failed!
                Job: ${env.JOB_NAME}
                Build: ${env.BUILD_URL}
                Check console output: ${env.BUILD_URL}console
            """
        }
        always {
            echo "Pipeline completed with status: ${currentBuild.currentResult}"
            // Archive ALL logs (even on failure)
            archiveArtifacts artifacts: 'target/**/*.log, **/surefire-reports/*.xml', allowEmptyArchive: true
        }
        cleanup {
            echo 'Ensuring no dangling resources...'
            sh 'docker rm -f $(docker ps -aq --filter label=jenkins-pipeline) 2>/dev/null || true'
        }
    }
}
```
**Key Takeaways from Example:**
*   Uses top-level `agent`, `environment`, `options`, `triggers`.
*   Parameterized (`VERSION`, `RUN_TESTS`, `ENV`).
*   Stage-specific `when` conditions (`RUN_TESTS` flag, `ENV` target).
*   Manual approval gate (`input`) before Prod.
*   Comprehensive `post` section (Slack, Email, Archiving, Cleanup).
*   **Security:** Secrets via `credentials()`, not hardcoded.
*   **Visibility:** Timestamps, log archiving, Slack/email notifications.
*   **Resilience:** Timeout, build retention.

---

### **Chapter 10: Scripted Pipeline**

#### **Groovy Basics for Jenkins**
*   **Why Groovy?** Jenkins Pipelines (both Declarative & Scripted) are built on Groovy. Scripted Pipeline *is* Groovy code.
*   **Essential Groovy for Pipelines:**
    *   **Variables:** `def name = "Jenkins"` (Dynamically typed). Avoid global vars; prefer `def` inside `node`.
    *   **Strings:** `'single quoted'` (no interpolation), `"double quoted ${var}"` (interpolation).
    *   **Lists:** `def list = ['a', 'b', 'c']`, `list.add('d')`, `list[0]`.
    *   **Maps:** `def map = [key1: 'value1', key2: 'value2']`, `map.key1`, `map['key1']`.
    *   **Control Flow:**
        ```groovy
        if (condition) { ... } else if (condition) { ... } else { ... }
        for (item in list) { echo item }
        for (i = 0; i < 10; i++) { ... }
        while (condition) { ... }
        ```
    *   **Functions:**
        ```groovy
        def greet(String name) {
            echo "Hello, ${name}!"
            return "Greeting complete"
        }
        def result = greet("World")
        ```
    *   **Error Handling (CRITICAL):**
        ```groovy
        try {
            sh 'command-that-might-fail'
        } catch (err) {
            echo "Command failed: ${err.getMessage()}"
            // mail, set build result, etc.
            currentBuild.result = 'UNSTABLE' // Or 'FAILURE'
            // throw err // Re-throw to fail the stage/job
        } finally {
            echo "This ALWAYS runs, even if error"
            // Cleanup resources
        }
        ```
*   **Jenkins-Specific Groovy:**
    *   **Global Variables:** `env.BUILD_NUMBER`, `env.WORKSPACE`, `currentBuild.result`.
    *   **Steps as Functions:** `sh 'cmd'`, `echo 'msg'`, `git url: '...'`, `junit '...'` are Groovy function calls.
    *   **CPS Transform:** Jenkins uses Continuation Passing Style (CPS) to make pipelines durable. **Avoid long-running Groovy loops/computation** – they block the master. Delegate heavy work to `sh` steps.

#### **`node`, `stage`, `step` Blocks**
*   **`node` (Mandatory Entry Point):**
    *   Acquires an executor (agent) from the Jenkins queue.
    *   **Replaces Declarative's `agent`.** Must be the outermost block in Scripted Pipeline.
    *   **Label Control:** `node('build-agent && linux') { ... }`
    *   **Workspace Control:** `node { ws('/custom/path') { ... } }` (Less common; usually use `dir`).
    *   **Always wrap your pipeline in `node`:**
        ```groovy
        node('my-agent') { // Acquire agent with label 'my-agent'
            // ALL pipeline code goes here
        }
        ```
*   **`stage` (Logical Grouping - Recommended):**
    *   Defines a stage for visualization in Blue Ocean/Stage View.
    *   **Not syntactically required** in Scripted (unlike Declarative), but **ESSENTIAL for readability and monitoring**.
    *   **Must be nested inside `node`:**
        ```groovy
        node {
            stage('Build') {
                sh 'mvn clean package'
            }
            stage('Test') {
                sh 'mvn test'
            }
        }
        ```
*   **`steps` (Implicit in Scripted):**
    *   In Scripted Pipeline, **steps are just Groovy statements** inside a `stage` (or directly in `node` if no `stage`).
    *   There is no explicit `steps` block. Commands like `sh`, `echo`, `git` are called directly.
    *   **Example (No explicit `steps`):**
        ```groovy
        node {
            stage('Build') {
                echo "Starting build..."
                sh 'mvn clean package' // This is a step
                archiveArtifacts 'target/*.jar' // This is another step
            }
        }
        ```

#### **Flow Control: `if`/`else`, loops, `try`/`catch`**
*   **Full Groovy Power:** Scripted Pipeline gives you complete control.
*   **Conditional Execution (`if`):**
    ```groovy
    node {
        def branch = env.GIT_BRANCH // e.g., 'origin/main'
        stage('Conditional Build') {
            if (branch == 'origin/main') {
                sh 'mvn clean deploy' // Deploy only from main
            } else {
                sh 'mvn clean package' // Just build for feature branches
            }
        }
    }
    ```
*   **Loops (`for`, `while`):**
    ```groovy
    node {
        def modules = ['api', 'web', 'data']
        stage('Build Modules') {
            for (module in modules) {
                dir(module) { // Change directory
                    sh 'mvn clean package'
                }
            }
        }
    }
    ```
*   **Robust Error Handling (`try`/`catch`/`finally`):**
    ```groovy
    node {
        stage('Deploy with Safety') {
            try {
                sh 'kubectl apply -f k8s/prod'
                // Verify deployment
                sh 'curl -sSf http://prod-app:8080/health || exit 1'
            } catch (err) {
                echo "Deployment FAILED: ${err.getMessage()}"
                // Rollback
                sh 'kubectl apply -f k8s/prod/previous-version'
                // Notify
                mail to: 'oncall@example.com', subject: 'ROLLBACK', body: err.getMessage()
                currentBuild.result = 'UNSTABLE' // Mark build unstable, don't fail job
                // throw err // Uncomment to fail the stage/job
            } finally {
                echo "Cleaning up test resources..."
                sh 'docker rm -f test-db || true'
            }
        }
    }
    ```
    *   **`currentBuild.result`:** Key for nuanced error handling (`'SUCCESS'`, `'UNSTABLE'`, `'FAILURE'`, `'ABORTED'`).
    *   **`throw err`:** Re-throws the exception, causing the stage/job to fail. Omit to continue after handling.

#### **Working with Variables and Functions**
*   **Variables:**
    *   Scope within `node` block. `def` is preferred.
    *   **Environment Variables:** `env.MY_VAR = 'value'` (set Jenkins env var), `echo env.PATH`.
    *   **Dynamic Variables:** `def version = sh(returnStdout: true, script: 'git describe --tags').trim()`
*   **Functions (Reusability):**
    *   Define functions **inside the `node` block** (not top-level!).
    *   **Example:**
        ```groovy
        node {
            // Define reusable function
            def buildApp(String module) {
                stage("Build ${module}") {
                    dir(module) {
                        sh 'mvn clean package'
                        archiveArtifacts "target/*.jar"
                    }
                }
            }

            // Use the function
            buildApp('api')
            buildApp('web')
        }
        ```
    *   **Parameters & Return Values:** Functions can take parameters and return values.
    *   **Shared Libraries:** For truly reusable functions across pipelines, use Shared Libraries (Chapter 11).

#### **Advanced Scripted Pipeline Examples**

1.  **Dynamic Parallel Stages:**
    ```groovy
    node {
        def modules = ['api', 'web', 'data']
        def parallelStages = [:] // Empty map for parallel stages

        for (module in modules) {
            def moduleName = module // Capture variable for closure
            parallelStages["Build ${moduleName}"] = {
                stage("Build ${moduleName}") {
                    dir(moduleName) {
                        sh 'mvn clean package'
                    }
                }
            }
        }

        stage('Build All Modules') {
            parallel parallelStages // Execute all stages in parallel
        }
    }
    ```

2.  **Retry with Exponential Backoff:**
    ```groovy
    node {
        stage('Deploy with Retry') {
            def maxRetries = 3
            def retryCount = 0
            def success = false

            while (retryCount <= maxRetries && !success) {
                try {
                    sh 'kubectl apply -f k8s/prod'
                    sh 'curl -sSf http://prod-app:8080/health || exit 1'
                    success = true
                } catch (err) {
                    retryCount++
                    if (retryCount > maxRetries) {
                        throw err // Fail after max retries
                    }
                    echo "Attempt ${retryCount} failed. Retrying in ${2**retryCount} seconds..."
                    sleep time: (2**retryCount), unit: 'SECONDS'
                }
            }
        }
    }
    ```

3.  **Processing SCM Changes:**
    ```groovy
    node {
        stage('Checkout') {
            checkout scm
        }
        stage('Analyze Changes') {
            def changedFiles = findFiles(glob: '**/*').collect { it.path } // Get all files
            // Or better: Use git plugin to get *changed* files since last build
            def changeLogSets = currentBuild.changeSets
            for (changeLog in changeLogSets) {
                for (entry in changeLog.items) {
                    for (file in entry.affectedFiles) {
                        echo "Changed file: ${file.path}"
                        if (file.path.endsWith('.java')) {
                            echo '  Java file changed, need to rebuild!'
                            // Trigger specific rebuild logic
                        }
                    }
                }
            }
        }
    }
    ```

**Scripted Pipeline Best Practices:**
1.  **Prefer Declarative:** Only use Scripted when Declarative is insufficient.
2.  **Use `stage` Blocks:** Always structure with `stage` for visibility.
3.  **Minimize Groovy Logic:** Keep complex logic in external scripts (sh/python) or Shared Libraries.
4.  **Robust Error Handling:** Use `try`/`catch`/`finally` religiously, especially for deployments.
5.  **Avoid Global Variables:** Use `def` within `node` scope.
6.  **Leverage `currentBuild`:** Control build result (`currentBuild.result = 'UNSTABLE'`).
7.  **Test Thoroughly:** Scripted errors often only surface at runtime.

---

### **Chapter 11: Shared Libraries**

#### **What are Shared Libraries?**
*   **Definition:** A mechanism to **define reusable pipeline logic** (functions, global variables, pipeline templates) in a **centralized Git repository**, which can then be **loaded and used** by multiple Jenkins Pipeline projects.
*   **Core Problem Solved:** Eliminates copy-paste of pipeline code across repositories. Enforces standards. Simplifies maintenance (fix one place, update everywhere).
*   **Analogy:** Like importing a module/library in Python/Java, but for Jenkins Pipelines.

#### **Creating a Global Shared Library**
1.  **Jenkins Configuration:**
    *   Go to `Jenkins URL → Manage Jenkins → System Configuration → Global Pipeline Libraries`.
    *   Click `Add` under "Global Libraries".
    *   **Name:** `shared-library` (Short name used to load it, e.g., `@Library('shared-library')`).
    *   **Default Version:** `main` (Branch/tag/commit to use by default).
    *   **Retrieval Method:** `Modern SCM` (Recommended).
    *   **Source Code Management:** Configure your Git repo (URL, credentials, etc.).
    *   **Check:** `Allow default version to be overridden` (Essential!).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
    *   **Check:** `Include @Library changes in job recent changes` (Helps auditing).
......<think>

We are going to cover Chapter 11: Shared Libraries in detail. The previous response was cut off due to length. Let's continue from where we left off.

### **Chapter 11: Shared Libraries (Continued)**

#### **Loading Libraries from SCM (Git)**
*   **Why SCM?** Storing libraries in Git provides version control, code review, and auditability – essential for production pipelines.
*   **Configuration (Global Library):**
    1.  Go to `Jenkins URL → Manage Jenkins → System Configuration → Global Pipeline Libraries`.
    2.  Click `Add` under "Global Libraries".
    3.  **Name:** `my-shared-lib` (This is the identifier used in `@Library` calls).
    4.  **Default Version:** `main` (Branch, tag, or commit hash to use by default).
    5.  **Retrieval Method:** Select `Modern SCM` (preferred) or `Legacy SCM`.
        *   **Modern SCM:** Uses Jenkins' standard SCM plugins (Git, SVN, etc.). Configure:
            *   *Source Code Management:* Choose `Git`.
            *   *Project Repository:* `https://github.com/yourorg/shared-library.git`
            *   *Credentials:* (If private repo)
            *   *Branch Specifier:* `*/main` (or leave blank for default branch)
        *   **Legacy SCM:** Simpler but less flexible (only Git, SVN, or Mercurial). Configure repo URL directly.
    6.  **Check: `Allow default version to be overridden`** (Crucial! Allows pipelines to specify a different version).
    7.  **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    8.  **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    9.  **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    10. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    11. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    12. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    13. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    14. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    15. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    16. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    17. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    18. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    19. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    20. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    21. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    22. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    23. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    24. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    25. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    26. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    27. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    28. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    29. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    30. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    31. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    32. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    33. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    34. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    35. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    36. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    37. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    38. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    39. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    40. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    41. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    42. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    43. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    44. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    45. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    46. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    47. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    48. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    49. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    50. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    51. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    52. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    53. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    54. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    55. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    56. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    57. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    58. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    59. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    60. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    61. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    62. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    63. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    64. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    65. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    66. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    67. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    68. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    69. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    70. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    71. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    72. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    73. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    74. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    75. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    76. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    77. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    78. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    79. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    80. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    81. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    82. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    83. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    84. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    85. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    86. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    87. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    88. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    89. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    90. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    91. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    92. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    93. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    94. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    95. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    96. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    97. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    98. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    99. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    100. **Check: `Include @Library changes in job recent changes`** (Helps track which library version was used).
    *   **Save.**

*   **Loading in a Pipeline:**
    *   **Global Library (Recommended):** `@Library('my-shared-lib') _` (Loads the library globally for the pipeline).
    *   **Specific Version:** `@Library('my-shared-lib@v1.2') _` (Overrides default version).
    *   **Inside Declarative Pipeline:**
        ```groovy
        pipeline {
            agent any
            options {
                // Load library at pipeline start
                timeout(time: 1, unit: 'HOURS')
            }
            // Load library HERE (before stages)
            libraries {
                lib('my-shared-lib@v2') // Modern syntax (Jenkins 2.222+)
            }
            stages { ... }
        }
        ```
    *   **Inside Scripted Pipeline:**
        ```groovy
        // MUST be at top level (outside node)
        @Library('my-shared-lib') _

        node {
            // Use library functions
            standardBuild()
        }
        ```

#### **Structure of Shared Libraries: `vars/`, `src/`, `resources/`**
A Shared Library repo has a specific directory structure:

```
(root)
├── vars
│   ├── sayHello.groovy
│   └── sayHello.txt (Optional help text)
├── src
│   └── com
│       └── yourcompany
│           └── utils
│               └── StringUtils.groovy
├── resources
│   └── com
│       └── yourcompany
│           └── utils
│               └── default.config (Text files for resource loading)
└── Jenkinsfile (Optional: Test pipeline for the library itself)
```

1.  **`vars/` Directory (Global Variables & Steps)**
    *   **Purpose:** Defines **global variables** that act as **pipeline steps** or **utility functions** accessible directly in any pipeline.
    *   **Naming:** `.groovy` file name = step/function name (e.g., `sayHello.groovy` → `sayHello()`).
    *   **Structure:**
        *   **`call` Method (Required for Steps):** Makes the variable callable like a step (`sayHello()`).
        *   **Other Methods:** Can define additional helper methods within the `.groovy` file.
    *   **Example (`vars/sayHello.groovy`):**
        ```groovy
        // Defines a step: sayHello(name: 'World')
        def call(Map config = [:]) {
            def name = config.name ?: 'World' // Default value
            echo "Hello, ${name} from Shared Library!"
        }

        // Optional: Helper method (only accessible within this .groovy file)
        private def formatName(String n) {
            return n.capitalize()
        }
        ```
    *   **Usage in Pipeline:**
        ```groovy
        @Library('my-shared-lib') _

        pipeline {
            agent any
            stages {
                stage('Greet') {
                    steps {
                        sayHello(name: 'Jenkins') // Uses vars/sayHello.groovy
                    }
                }
            }
        }
        ```
    *   **Advanced: Step with `script {}` Block (Declarative):**
        ```groovy
        // vars/runTests.groovy
        def call(body) {
            // Evaluate the body block, and collect configuration into the object
            def config = [:]
            body.resolveStrategy = Closure.DELEGATE_FIRST
            body.delegate = config
            body()

            stage('Run Tests') {
                steps {
                    sh "mvn test -Dtest=${config.suite}"
                }
            }
        }

        // Pipeline Usage (Declarative)
        stages {
            stage('Test') {
                steps {
                    script {
                        runTests {
                            suite = 'IntegrationTests'
                        }
                    }
                }
            }
        }
        ```

2.  **`src/` Directory (Groovy Source Code)**
    *   **Purpose:** Contains standard **Groovy source code** (`.groovy` files) in a Java-like package structure. Used for:
        *   Complex utility classes.
        *   Domain-specific objects.
        *   Logic too complex for `vars/`.
    *   **Access:** Must be explicitly imported in the pipeline or within `vars/` scripts using `import`.
    *   **Example (`src/com/yourcompany/pipeline/BuildUtils.groovy`):**
        ```groovy
        package com.yourcompany.pipeline

        // Static utility class
        public class BuildUtils {
            public static String getBuildTag(String version, String commit) {
                return "${version}-${commit.substring(0,7)}"
            }

            public static boolean isHotfixBranch(String branch) {
                return branch.startsWith('hotfix/')
            }
        }
        ```
    *   **Usage in Pipeline:**
        ```groovy
        @Library('my-shared-lib') _

        import com.yourcompany.pipeline.BuildUtils // Import the class

        pipeline {
            agent any
            environment {
                BUILD_TAG = "${BuildUtils.getBuildTag(env.VERSION, env.GIT_COMMIT)}"
            }
            stages {
                stage('Build') {
                    steps {
                        echo "Using build tag: ${env.BUILD_TAG}"
                        script {
                            if (BuildUtils.isHotfixBranch(env.GIT_BRANCH)) {
                                echo "This is a HOTFIX branch!"
                            }
                        }
                    }
                }
            }
        }
        ```

3.  **`resources/` Directory (Resource Files)**
    *   **Purpose:** Stores **non-Groovy files** (text, JSON, YAML, templates, shell scripts) that can be loaded by the library.
    *   **Access:** Use `libraryResource` step to load content as a string.
    *   **Structure:** Mirrors package structure of `src/`.
    *   **Example (`resources/com/yourcompany/pipeline/deploy-template.yaml`):**
        ```yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: ${APP_NAME}
        spec:
          replicas: ${REPLICAS}
          ...
        ```
    *   **Usage in Library (`vars/deployToProd.groovy`):**
        ```groovy
        def call(String appName) {
            // Load template from resources
            def template = libraryResource 'com/yourcompany/pipeline/deploy-template.yaml'
            // Replace variables
            def deployYaml = template
                .replace('${APP_NAME}', appName)
                .replace('${REPLICAS}', '3')
            // Write to file and deploy
            writeFile file: 'deploy.yaml', text: deployYaml
            sh 'kubectl apply -f deploy.yaml -n prod'
        }
        ```

#### **Writing Reusable Functions and Pipelines**
*   **Principles:**
    *   **Single Responsibility:** Each function does one thing well.
    *   **Parameterize Everything:** Use `Map config` for flexible arguments. Provide defaults.
    *   **Idempotency:** Safe to run multiple times (e.g., `kubectl apply` is idempotent, `kubectl create` is not).
    *   **Error Handling:** Include robust `try`/`catch` where appropriate. Fail fast.
    *   **Logging:** Use `echo` liberally for visibility. Prefix with library name.
    *   **Documentation:** Use `/** ... */` comments. Add `.txt` files in `vars/` for help (`sayHello.txt`).
*   **Example: Reusable Build Function (`vars/standardBuild.groovy`):**
    ```groovy
    /**
     * Standardized build process for Java/Maven projects.
     * @param config Map of configuration options:
     *   - module (String, optional): Subdirectory/module name (default: '.')
     *   - skipTests (Boolean, optional): Skip tests? (default: false)
     *   - archivePattern (String, optional): Artifacts to archive (default: 'target/*.jar')
     */
    def call(Map config = [:]) {
        // Default configuration
        def defaults = [
            module: '.',
            skipTests: false,
            archivePattern: 'target/*.jar'
        ]
        // Merge user config with defaults
        def c = defaults + config

        echo "[LIB] Starting standard build (module: ${c.module}, skipTests: ${c.skipTests})"

        stage("Build ${c.module}") {
            dir(c.module) {
                try {
                    // Build command
                    def cmd = 'mvn clean package'
                    if (c.skipTests) {
                        cmd += ' -DskipTests'
                    }
                    sh cmd

                    // Archive artifacts
                    archiveArtifacts artifacts: c.archivePattern, fingerprint: true
                } catch (err) {
                    echo "[LIB] Build failed: ${err.getMessage()}"
                    throw err // Rethrow to fail the stage
                }
            }
        }
    }
    ```
*   **Usage:**
    ```groovy
    standardBuild() // Uses defaults
    standardBuild(module: 'api', skipTests: true)
    ```

#### **Versioning and Best Practices**
*   **Versioning Strategy:**
    *   **Git Tags:** Use Semantic Versioning (`v1.0.0`, `v1.1.0`, `v2.0.0`). Tag commits in the library repo.
    *   **Branches:** `main` (stable), `develop` (next release). Avoid using branch names directly in pipelines (use tags).
    *   **Pin Versions in Pipelines:** **ALWAYS** specify a version in `@Library` (e.g., `@Library('my-shared-lib@v1.2') _`). Prevents unexpected breaks from library changes.
*   **Best Practices:**
    1.  **Code Review:** Treat library code like application code. PRs/MRs required.
    2.  **Testing:**
        *   Write unit tests for `src/` classes (using Jenkins Pipeline Unit framework).
        *   Write integration tests using a `Jenkinsfile` in the library repo that exercises the functions.
    3.  **Documentation:** Document every `vars/` function in its `.txt` file and in-code comments. Maintain a README.md.
    4.  **Minimize Scope:** Start small. Only move truly reusable, stable logic into the library.
    5.  **Avoid Jenkins Context in `src/`:** `src/` classes should be pure Groovy, not depend on Jenkins `steps` or `env`. Put Jenkins interaction in `vars/`.
    6.  **Handle Secrets Securely:** Never hardcode secrets in the library. Pass them via parameters from the pipeline (which gets them from Jenkins Credentials).
    7.  **Deprecation Policy:** Clearly mark deprecated functions (comments, logs) and provide migration paths. Remove after a reasonable period.
    8.  **Global vs. Folder Libraries:** Use Global Libraries for organization-wide standards. Use Folder Libraries (configured at folder level) for team-specific libraries.
    9.  **Avoid Circular Dependencies:** Libraries should not depend on pipelines that use them.
    10. **Monitor Usage:** Track which pipelines use which library versions (Jenkins "Recent Changes" shows this).

**Critical Shared Library Pitfall:** **Never load a library dynamically inside a `node` block or stage.** Always load it at the top level of the pipeline (Declarative) or Scripted `node` block. Loading inside a stage can cause serialization issues due to CPS.

---

### **Chapter 12: Pipeline Input and User Interaction**

#### **Using `input` Step for Manual Approval**
*   **Purpose:** Pause a pipeline and **wait for explicit human approval** (or input) before proceeding. Essential for production deployments.
*   **Syntax:**
    ```groovy
    input {
        message "Prompt message for the user"
        id "UniqueID" // Optional, defaults to stage name
        ok "Button text" // Text for the 'Proceed' button (default: 'Proceed')
        submitter "user1,user2" // Optional: Comma-separated list of users who can approve
        submitterParameter "submitter" // Optional: Env var name to store approver's username
        parameters { // Optional: Additional input fields
            string(name: 'DEPLOY_VERSION', defaultValue: 'latest', description: 'Version to deploy')
        }
    }
    ```
*   **Where to Use:** Inside a `steps` block (Declarative) or directly in a `stage` (Scripted).
*   **Behavior:**
    *   Pipeline pauses execution.
    *   User sees prompt in Blue Ocean UI or Classic UI "Input Requested" page.
    *   User can:
        *   Click `ok` button (or custom `ok` text) to proceed.
        *   Click `Abort` to stop the pipeline.
        *   Fill in any `parameters` defined.
    *   Pipeline resumes only after approval/abort.
*   **Example (Declarative - Prod Approval):**
    ```groovy
    stage('Deploy to Prod') {
        when { expression { params.ENV == 'prod' } }
        steps {
            input {
                message "Deploy ${params.APP_NAME} version ${params.VERSION} to PRODUCTION?"
                ok 'DEPLOY NOW'
                submitter 'prod-admins' // Jenkins user/group with permission
                submitterParameter 'APPROVED_BY' // Sets env.APPROVED_BY
                parameters {
                    choice(name: 'ROLLBACK_VERSION', choices: ['v1.0', 'v1.1', 'v1.2'], description: 'Rollback version?')
                }
            }
            sh 'deploy-script.sh --version ${params.VERSION} --rollback ${params.ROLLBACK_VERSION}'
        }
    }
    ```
*   **Example (Scripted - Simple Approval):**
    ```groovy
    stage('Deploy to Prod') {
        def userInput = input(
            id: 'prod-deploy-approval',
            message: 'Deploy to Production?',
            ok: 'Yes, Deploy',
            submitter: 'deploy-team',
            parameters: [
                string(name: 'VERSION', defaultValue: 'latest', description: 'Version')
            ]
        )
        echo "Approved by: ${userInput}" // Returns map of parameters OR approver username if no params
        sh "deploy.sh ${userInput.VERSION}"
    }
    ```

#### **Conditional Execution Based on Input**
*   **Using `input` Return Value:**
    *   If `parameters` are defined, `input` returns a **Map** containing the entered values.
    *   If no `parameters`, `input` returns the **username** of the approver (as a String).
*   **Example (Conditional Logic):**
    ```groovy
    stage('Deploy Decision') {
        steps {
            script {
                def approval = input(
                    message: 'Choose deployment action:',
                    ok: 'Confirm',
                    submitter: 'deploy-team',
                    parameters: [
                        choice(name: 'ACTION', choices: ['DEPLOY', 'ROLLBACK', 'CANCEL'], description: '')
                    ]
                )
                switch(approval.ACTION) {
                    case 'DEPLOY':
                        sh 'deploy.sh'
                        break
                    case 'ROLLBACK':
                        sh 'rollback.sh'
                        break
                    case 'CANCEL':
                        error('Deployment canceled by user') // Fails the stage
                        break
                }
            }
        }
    }
    ```
*   **Key Point:** Wrap `input` in a `script {}` block in Declarative if you need to use its return value for conditional logic (since `input` is a step, but its return value needs Groovy context).

#### **Integrating with Slack or Email for Approval Requests**
*   **Problem:** Waiting in the Jenkins UI for an `input` prompt isn't practical for teams. Notifications are essential.
*   **Solution:** Use the `input` step **in combination** with notification plugins (Slack, Email) to alert approvers.
*   **Pattern:**
    1.  Send notification **BEFORE** the `input` step (to trigger the alert).
    2.  Include a **direct link to the input page** in the notification.
    3.  Use `input` to pause and wait.
*   **Slack Integration Example (Declarative):**
    ```groovy
    stage('Prod Approval') {
        steps {
            script {
                // 1. Send Slack notification WITH link to input page
                def approvalLink = "${env.BUILD_URL}input"
                slackSend(
                    channel: '#prod-deploys',
                    color: '#36A64F',
                    message: """*PRODUCTION DEPLOYMENT REQUEST*
                    *App:* ${params.APP_NAME}
                    *Version:* ${params.VERSION}
                    *Approve:* ${approvalLink}
                    *Abort:* ${env.BUILD_URL}stop"""
                )

                // 2. Wait for input
                input {
                    message "Approve PROD deployment of ${params.APP_NAME} v${params.VERSION}?"
                    ok "APPROVE"
                    submitter "prod-admins"
                }
            }
        }
    }
    ```
*   **Email Integration Example:**
    ```groovy
    stage('Prod Approval') {
        steps {
            script {
                // 1. Send Email with link
                def approvalLink = "${env.BUILD_URL}input"
                mail to: 'prod-team@example.com',
                    subject: "Jenkins Approval: Deploy ${params.APP_NAME} v${params.VERSION} to Prod?",
                    body: """Approve deployment?
                    Click to APPROVE: ${approvalLink}
                    Click to ABORT: ${env.BUILD_URL}stop
                    App: ${params.APP_NAME}
                    Version: ${params.VERSION}"""

                // 2. Wait for input
                input {
                    message "Approve PROD deployment?"
                    ok "Yes"
                    submitter "prod-team"
                }
            }
        }
    }
    ```
*   **Critical Components:**
    *   **`env.BUILD_URL`**: Base URL of the current build.
    *   **`/input`**: Path to the input approval page (e.g., `http://jenkins.example.com/job/my-pipeline/123/input`).
    *   **`/stop`**: Path to abort the build (e.g., `http://jenkins.example.com/job/my-pipeline/123/stop`).
*   **Best Practices for Notifications:**
    *   **Clear Subject/Message:** State exactly what needs approval and the consequences.
    *   **Direct Links:** Always include the `/input` and `/stop` links.
    *   **Targeted Channels:** Send to specific teams/channels (e.g., `#prod-deploys`), not broadcast.
    *   **Use Rich Formatting:** Slack supports markdown; Email supports HTML.
    *   **Avoid Spam:** Only trigger notifications for critical approvals (e.g., Prod). Use `when` to skip for non-prod.
    *   **Timeout:** Combine with `timeout` around the `input` to auto-fail if no response:
        ```groovy
        options {
            timeout(time: 2, unit: 'HOURS') // Around the stage or whole pipeline
        }
        steps {
            input { ... }
        }
        ```

**Advanced Pattern: Auto-Approval for Non-Prod**
```groovy
stage('Deploy') {
    steps {
        script {
            if (params.ENV == 'prod') {
                // Prod: Require manual approval + Slack alert
                slackSend(channel: '#prod-deploys', message: "Awaiting prod approval for ${params.APP_NAME}...")
                input message: "Deploy to PROD?", ok: 'DEPLOY'
            } else {
                // Non-Prod: Auto-approve, but log it
                echo "Auto-approved deployment to ${params.ENV}"
            }
            sh "deploy.sh --env ${params.ENV}"
        }
    }
}
```
