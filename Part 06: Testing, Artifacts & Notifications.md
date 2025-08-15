### **Chapter 17: Integrating Testing in Jenkins**

#### **1. Running Unit Tests (JUnit, TestNG, pytest, etc.)**
*   **What it is**: Executing automated tests during the build to validate code functionality.
*   **Why**: Fail fast, prevent broken code from progressing, ensure quality gates.
*   **How it Works**:
    *   Jenkins executes test commands (e.g., `mvn test`, `pytest`, `gradle test`) as part of the build.
    *   Tests must output results in a machine-readable format (typically XML - JUnit/XUnit format).
*   **Configuration**:
    *   **Freestyle Project**:
        1.  *Build* section → Add build step → "Execute shell" / "Windows Batch Command".
        2.  Command: `mvn clean test` (Maven), `pytest --junit-xml=test-results.xml` (pytest), `gradle test` (Gradle).
        3.  *Crucial*: Ensure test frameworks are configured to generate XML reports (e.g., pytest's `--junit-xml`, Maven Surefire/Failsafe plugins).
    *   **Declarative Pipeline**:
        ```groovy
        pipeline {
            agent any
            stages {
                stage('Test') {
                    steps {
                        sh 'mvn clean test' // Or 'pytest --junit-xml=test-results.xml'
                    }
                }
            }
        }
        ```
*   **Key Plugins & Frameworks**:
    *   **JUnit**: Default for Java (Maven: Surefire plugin, Gradle: built-in).
    *   **TestNG**: Java alternative (Maven: Surefire plugin with `testng` dependency).
    *   **pytest**: Python (Use `--junit-xml` flag).
    *   **Mocha/Chai/Jest**: JavaScript (Use reporters like `mocha-junit-reporter`, `jest-junit`).
    *   **NUnit/xUnit**: .NET (Use `nunit3-console` with `--result` flag, `dotnet test --logger trx`).
*   **Critical Best Practices**:
    *   **Isolate Tests**: Run tests in a clean environment (use Docker agents or clean workspaces).
    *   **Parallelize**: Split test suites (e.g., `mvn -T 4C test`, pytest `-n auto`).
    *   **Flaky Tests**: Use retries (e.g., TestNG `retryAnalyzer`, pytest `--reruns 2`).
    *   **Test Data**: Use in-memory DBs (H2) or reset databases before/after tests.
*   **Pitfalls**:
    *   **Missing XML Reports**: Jenkins won't find results. *Fix:* Verify test command output path.
    *   **Test Failures Ignored**: Build succeeds despite test failures. *Fix:* Ensure test commands return non-zero exit code on failure (most frameworks do this by default).
    *   **Environment Dependencies**: Tests pass locally but fail on Jenkins. *Fix:* Containerize tests or replicate environment exactly.

#### **2. Publishing Test Results (JUnit Plugin)**
*   **What it is**: Jenkins plugin (` junit`) to parse XML test results, display trends, and fail builds.
*   **Why**: Visualize pass/fail rates, identify flaky tests, enforce quality gates.
*   **Configuration**:
    *   **Install Plugin**: Manage Jenkins → Plugins → Available → "JUnit" → Install.
    *   **Freestyle Project**:
        1.  *Post-build Actions* → "Publish JUnit test result report".
        2.  *Test report XMLs*: `**/target/surefire-reports/*.xml` (Maven), `test-results.xml` (pytest), `**/TEST-*.xml` (common pattern).
        3.  *Optional*: Set thresholds for "Unstable" (e.g., >5% failures) or "Failed" (e.g., >0% failures).
    *   **Declarative Pipeline**:
        ```groovy
        post {
            always {
                junit testResults: '**/target/surefire-reports/*.xml', 
                      keepLongStdio: true,
                      healthScaleFactor: 1.0,
                      allowEmptyResults: false // Fails build if no reports found
            }
        }
        ```
*   **Key Features**:
    *   **Trend Graph**: Shows pass/fail rates over builds.
    *   **Test Result Details**: Drill down into failed tests, view stack traces.
    *   **Flaky Test Detection**: Marks tests that pass/fail inconsistently.
    *   **Thresholds**: `Unstable` (warning), `Failed` (build failure) based on failure %.
*   **Advanced Settings**:
    *   `keepLongStdio`: Preserves stdout/stderr for failed tests (essential for debugging).
    *   `skipIfNoTestFiles`: Avoids build failure if tests weren't run (e.g., skipped stage).
    *   `testDataPublishers`: Attach screenshots/videos for failed UI tests (requires extra setup).
*   **Troubleshooting**:
    *   **"No test reports found"**: Check glob pattern, workspace paths, plugin installation.
    *   **"Invalid XML"**: Validate report format (use `xmllint`), ensure test framework outputs JUnit schema.
    *   **Build Not Failing**: Verify `allowEmptyResults: false` and thresholds.

#### **3. Code Coverage with JaCoCo, Cobertura**
*   **What it is**: Measuring % of code executed by tests (statements, branches, lines).
*   **Why**: Identify untested code, enforce coverage minimums.
*   **JaCoCo (Java)**:
    *   **How it Works**: Java agent instruments bytecode at runtime, generates `.exec` files, converted to XML/HTML.
    *   **Setup**:
        1.  **Maven**: Add plugin to `pom.xml`:
            ```xml
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.8.11</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>prepare-agent</goal>
                            <goal>report</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            ```
        2.  **Jenkins**:
            *   *Post-build Action*: "Record JaCoCo coverage report".
            *   *Pattern*: `**/target/site/jacoco/jacoco.xml`
            *   *Thresholds*: Set minimum coverage for builds to be stable/unstable/failed.
    *   **Pipeline**:
        ```groovy
        steps {
            sh 'mvn clean test'
            step([$class: 'JacocoPublisher', 
                  execPattern: '**/target/jacoco.exec',
                  classPattern: '**/target/classes',
                  sourcePattern: '**/src/main/java',
                  exclusionPattern: '**/src/test*',
                  minimumInstructionCoverage: '80',
                  minimumBranchCoverage: '70'])
        }
        ```
*   **Cobertura (Multi-Language)**:
    *   **How it Works**: Parses coverage data from frameworks (e.g., `coverage.py` for Python).
    *   **Setup**:
        1.  **Python Example**:
            ```bash
            pip install coverage
            coverage run -m pytest
            coverage xml  # Generates cobertura.xml
            ```
        2.  **Jenkins**:
            *   *Post-build Action*: "Publish Cobertura Coverage Report".
            *   *Pattern*: `**/cobertura.xml`
    *   **Pipeline**:
        ```groovy
        step([$class: 'CoberturaPublisher', 
              autoUpdateHealth: false,
              autoUpdateStability: false,
              coberturaReportFile: '**/cobertura.xml',
              healthyTarget: [methodCoverage: 80, lineCoverage: 85]])
        ```
*   **Critical Best Practices**:
    *   **Set Realistic Thresholds**: Start low (e.g., 50%), incrementally increase.
    *   **Track Trends**: Use Jenkins trend graphs to monitor coverage over time.
    *   **Exclude Generated Code**: (e.g., Lombok, Protobuf) via plugin settings or config files.
    *   **Merge Reports**: For multi-module projects, use `jacoco:merge` or `coverage combine`.
*   **Pitfalls**:
    *   **Agent Not Attached**: JaCoCo requires JVM arg `-javaagent:jacoco.jar=destfile=coverage.exec`. Maven/Gradle plugins handle this.
    *   **Stale Reports**: Clean `target/` directory before builds.
    *   **Inaccurate Branch Coverage**: Complex logic (e.g., ternary ops) may not be fully covered.

#### **4. Static Code Analysis (SonarQube, PMD, Checkstyle)**
*   **What it is**: Analyzing source code *without* execution to find bugs, vulnerabilities, style issues.
*   **Why**: Enforce coding standards, detect bugs early, improve maintainability.
*   **SonarQube (Gold Standard)**:
    *   **How it Works**: Jenkins sends code to SonarQube server for analysis; results displayed in Jenkins & SonarQube UI.
    *   **Setup**:
        1.  Install SonarQube server + Scanner plugin in Jenkins.
        2.  Configure *Manage Jenkins → Configure System → SonarQube servers* (Add server URL & token).
        3.  **Freestyle**:
            *   *Build* → "SonarQube Scanner" → Select server, set project key/name.
            *   *Advanced*: `sonar.sources=src`, `sonar.java.binaries=target/classes`.
        4.  **Pipeline (Declarative)**:
            ```groovy
            stage('SonarQube Analysis') {
                steps {
                    withSonarQubeEnv('MySonarServer') { // Configured server name
                        sh 'mvn sonar:sonar -Dsonar.projectKey=my-project'
                    }
                }
            }
            // Optional: Wait for Quality Gate
            stage('Quality Gate') {
                steps {
                    timeout(time: 1, unit: 'HOURS') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
            ```
    *   **Key Features**: Quality Gates (pass/fail builds), Security Hotspots, Technical Debt tracking.
*   **PMD & Checkstyle (Lightweight)**:
    *   **PMD**: Finds bugs, suboptimal code, unused code (Java, JS, etc.).
    *   **Checkstyle**: Enforces coding style (indentation, naming, etc.).
    *   **Configuration**:
        1.  Install "PMD Plugin" / "Checkstyle Plugin".
        2.  **Freestyle** → *Post-build Actions* → "Scan for compiler warnings" → Select tool.
        3.  *Pattern*: `**/pmd.xml`, `**/checkstyle-result.xml`.
        4.  Set health thresholds (e.g., "Unstable if >100 warnings").
    *   **Pipeline**:
        ```groovy
        post {
            always {
                recordIssues tools: [pmd(pattern: '**/pmd.xml'), checkstyle(pattern: '**/checkstyle-result.xml')]
            }
        }
        ```
*   **Best Practices**:
    *   **Start Small**: Enable only critical rules initially.
    *   **Integrate with PRs**: Use SonarQube's "Pull Request Decoration".
    *   **Baseline Existing Code**: Analyze current state before enforcing new rules.
    *   **Use Quality Gates**: Block merges if coverage/security issues exceed thresholds.
*   **Pitfalls**:
    *   **False Positives**: Tune rulesets; don't ignore valid warnings.
    *   **Slow Analysis**: Exclude `node_modules`, `target`, `venv` directories.
    *   **Token Permissions**: SonarQube token needs "Execute Analysis" permission.

#### **5. Security Scanning (OWASP ZAP, Snyk, Trivy)**
*   **What it is**: Automated tools to find security vulnerabilities in code, dependencies, and containers.
*   **Why**: Prevent critical vulnerabilities (e.g., Log4j), comply with regulations.
*   **OWASP ZAP (Dynamic Application Security Testing - DAST)**:
    *   **How it Works**: Actively scans running web apps for vulnerabilities (XSS, SQLi).
    *   **Setup**:
        1.  Install "ZAP Jenkins Plugin".
        2.  Start target app (e.g., `docker-compose up -d`).
        3.  **Freestyle** → *Build* → "Execute ZAP Scan":
            *   *Target URL*: `http://app:8080`
            *   *Scan Type*: "Quick Scan" or "Full Scan"
            *   *Fail Build*: On high-risk alerts.
        4.  **Pipeline**:
            ```groovy
            stage('ZAP Scan') {
                steps {
                    zap(site: 'http://app:8080', 
                        rules: 'rules.txt', 
                        failHigh: true, 
                        failMedium: false)
                }
            }
            ```
    *   **Best Practice**: Run against staging environment; use API scan for APIs.
*   **Snyk (Software Composition Analysis - SCA)**:
    *   **How it Works**: Scans dependencies (`package.json`, `pom.xml`, `requirements.txt`) for known vulns.
    *   **Setup**:
        1.  Get Snyk API token (`snyk auth`).
        2.  **Freestyle** → *Build* → "Execute shell": `snyk test --json > snyk-results.json`
        3.  *Post-build* → "Snyk Security Issues" → `snyk-results.json`.
        4.  **Pipeline**:
            ```groovy
            steps {
                sh 'snyk test --json > snyk-results.json'
                snykSecurity resultsFile: 'snyk-results.json', failOnIssues: true, severity: 'high'
            }
            ```
*   **Trivy (Container/Image Scanning)**:
    *   **How it Works**: Scans Docker images for OS packages & language-specific dependencies.
    *   **Setup**:
        1.  Build image: `docker build -t myapp:$BUILD_ID .`
        2.  **Pipeline**:
            ```groovy
            stage('Trivy Scan') {
                steps {
                    sh 'trivy image --format json --output trivy-report.json myapp:$BUILD_ID'
                    recordIssues tool: trivyParser(pattern: 'trivy-report.json')
                }
            }
            ```
    *   **Critical**: Scan *before* pushing to registry; fail on critical CVEs.
*   **Best Practices**:
    *   **Shift Left**: Scan in PRs, not just main branch.
    *   **Prioritize Severity**: Fail builds only on Critical/High vulns initially.
    *   **Update Baselines**: Regularly update tool databases (Snyk CLI, Trivy DB).
    *   **Combine Tools**: Use Snyk for dependencies + Trivy for containers + ZAP for runtime.
*   **Pitfalls**:
    *   **Noise**: Too many low-sev issues → ignore critical ones. *Fix:* Triage results.
    *   **False Negatives**: Tools miss vulns. *Fix:* Combine with manual pentests.
    *   **Token Security**: Store tokens in Jenkins Credentials, *never* in code.

---

### **Chapter 18: Artifact Management**

#### **1. Archiving Build Artifacts**
*   **What it is**: Saving build outputs (JARs, WARs, binaries, reports) to Jenkins master.
*   **Why**: Retain outputs for debugging, auditing, or downstream jobs.
*   **Configuration**:
    *   **Freestyle**:
        1.  *Post-build Actions* → "Archive the artifacts".
        2.  *Files to archive*: `target/*.jar, build/reports/**` (Ant-style glob).
        3.  *Optional*: "Only if build is successful", "Fingerprint files".
    *   **Declarative Pipeline**:
        ```groovy
        post {
            success {
                archiveArtifacts artifacts: 'target/*.jar, build/reports/**', 
                                fingerprint: true, 
                                allowEmptyArchive: false
            }
        }
        ```
*   **Key Options**:
    *   **Fingerprinting**: Tracks which artifacts were used in downstream builds (enables traceability).
    *   **Exclusions**: Use `!target/exclude-me.jar` in glob pattern.
    *   **Retention**: Configure in *Job Settings → Discard Old Builds* (e.g., keep last 10 builds with artifacts).
*   **Pitfalls**:
    *   **Disk Space**: Jenkins master fills up. *Fix:* Use short retention, archive only essentials.
    *   **Large Artifacts**: Slows down Jenkins. *Fix:* Push to Artifactory/Nexus instead.
    *   **Path Errors**: Use absolute paths in pipeline (`pwd() + '/target/*.jar'`).

#### **2. Publishing to Nexus, Artifactory, or S3**
*   **Why**: Centralized, scalable storage with metadata (vs. Jenkins archiving). Enables sharing across teams.
*   **Nexus/Artifactory (Binary Repositories)**:
    *   **Setup**:
        1.  Install "Artifactory Plugin" or "Nexus Platform Plugin".
        2.  *Manage Jenkins → Configure System* → Add server credentials (username/password or API key).
    *   **Publishing (Artifactory Example)**:
        ```groovy
        stage('Deploy to Artifactory') {
            steps {
                rtUpload (
                    serverId: 'artifactory-server', // Configured server ID
                    spec: '''{
                        "files": [
                            {
                                "pattern": "target/*.jar",
                                "target": "libs-release-local/com/example/"
                            }
                        ]
                    }''',
                    failNoOp: true
                )
            }
        }
        ```
    *   **Key Features**: Versioning, metadata (build info), access control, replication.
*   **S3 (Amazon Simple Storage Service)**:
    *   **Setup**:
        1.  Install "AWS Credentials" and "S3 Plugin".
        2.  Add AWS credentials (IAM user with `s3:PutObject` permissions).
    *   **Publishing**:
        ```groovy
        step([$class: 'S3BucketPublisher',
              profileName: 'my-aws-profile', // Configured in Jenkins
              entries: [
                  [sourceFile: 'target/*.jar',
                   destination: "my-bucket/releases/${BUILD_ID}/",
                   noUploadOnFailure: true]
              ]])
        ```
*   **Best Practices**:
    *   **Immutable Artifacts**: Use unique build IDs (e.g., `app-${BUILD_ID}.jar`).
    *   **Promotion Workflow**: Deploy to "snapshot" repo on dev branch, "release" repo on main.
    *   **Cleanup**: Use repository retention policies (e.g., Artifactory "Delete Old Snapshots").
*   **Pitfalls**:
    *   **Permission Errors**: Verify IAM/Artifactory permissions.
    *   **Overwriting**: Use unique paths (include `${BUILD_ID}`).
    *   **Network Issues**: Handle retries in pipeline (e.g., `retry(3)` block).

#### **3. Using Artifactory Plugin**
*   **Why**: Deep integration with JFrog Artifactory (superior to generic S3/Nexus for build-info).
*   **Key Features**:
    *   **Build-Info**: Captures *all* artifacts, dependencies, and environment used in the build.
    *   **Resolve Dependencies**: Download dependencies from Artifactory during build.
    *   **Xray Integration**: Trigger vulnerability scans on deployed artifacts.
*   **Advanced Configuration**:
    ```groovy
    def server = Artifactory.server 'artifactory-server'
    def rtMaven = Artifactory.newMavenBuilder(server)

    stage('Build & Deploy') {
        steps {
            rtMaven.run pom: 'pom.xml',
                        goals: 'clean install',
                        buildInfo: buildInfo // Captures build info
        }
    }
    stage('Publish Build Info') {
        steps {
            server.publishBuildInfo buildInfo // Sends full build metadata to Artifactory
        }
    }
    ```
*   **Critical Benefits**:
    *   **Traceability**: See *exactly* which dependencies were used in a build.
    *   **Reproducibility**: Rebuild any past version with full dependency graph.
    *   **Impact Analysis**: Xray shows which apps are affected by a CVE.

#### **4. Downloading Artifacts in Downstream Jobs**
*   **Why**: Pass artifacts between jobs (e.g., build → test → deploy).
*   **Methods**:
    *   **1. Jenkins Artifact Archiving**:
        ```groovy
        // Upstream Job
        archiveArtifacts 'target/app.jar'

        // Downstream Job
        copyArtifacts(projectName: 'upstream-job', filter: 'target/app.jar')
        ```
    *   **2. Artifactory/Nexus** (Recommended for large artifacts):
        ```groovy
        // Upstream: Push to repo (as shown in Section 2)
        // Downstream: Resolve from repo
        rtDownload (
            serverId: 'artifactory-server',
            spec: '''{
                "files": [{
                    "pattern": "libs-release-local/com/example/app-*.jar",
                    "target": "lib/"
                }]
            }'''
        )
        ```
    *   **3. S3**:
        ```groovy
        step([$class: 'S3BucketPublisher',
              entries: [[sourceFile: 'lib/app.jar', ...]]]) // Upstream
        step([$class: 'S3BucketPublisher',
              entries: [[sourceFile: '', destination: 'lib/', ...]]]) // Downstream
        ```
*   **Best Practices**:
    *   **Avoid `copyArtifacts` for large files**: Uses Jenkins master → network bottleneck.
    *   **Use Repositories**: Artifactory/Nexus are designed for this.
    *   **Fingerprinting**: Verify artifact integrity between jobs.

#### **5. Clean Workspace Strategies**
*   **Why**: Prevent "ghost" files from previous builds causing failures.
*   **Methods**:
    *   **1. `cleanWs()` (Pipeline)**:
        ```groovy
        stage('Clean') {
            steps {
                cleanWs notFailBuild: true, // Doesn't fail build if cleanup fails
                     deleteDirs: true,      // Deletes directories (not just files)
                     disableDeferredWipe: true,
                     exclude: ['important.log'] // Keep specific files
            }
        }
        ```
    *   **2. Freestyle Project**:
        *   *Build Environment* → "Delete workspace before build starts".
    *   **3. Advanced Cleanup**:
        *   **`ws-cleanup` Plugin**: More granular control (e.g., `cleanBefore: true`, `cleanAfter: true`).
        *   **Custom Script**:
            ```bash
            # Delete all except .git and logs
            find . -mindepth 1 ! -name '.git' ! -name 'logs' -exec rm -rf {} +
            ```
*   **Strategies**:
    *   **Pre-Build Cleanup**: Essential for reproducibility (default recommended).
    *   **Post-Build Cleanup**: Save disk space; use `cleanWs()` in `post` block.
    *   **Selective Cleanup**: Keep logs, cache directories (e.g., `~/.m2` for Maven).
*   **Pitfalls**:
    *   **Accidental Deletion**: Test cleanup scripts thoroughly.
    *   **Performance**: `cleanWs()` slows down builds. *Fix:* Exclude large caches.
    *   **Agent-Specific**: Use `cleanWs()` instead of `sh 'rm -rf *'` for Windows/Linux compatibility.

---

### **Chapter 19: Notifications and Reporting**

#### **1. Email Notifications (SMTP Setup)**
*   **Why**: Alert teams on build status (success/failure).
*   **Setup**:
    1.  *Manage Jenkins → Configure System* → **Extended E-mail Notification**:
        *   SMTP server: `smtp.gmail.com` (for Gmail), `smtp.office365.com` (for Outlook)
        *   SMTP Port: `587` (TLS) or `465` (SSL)
        *   Credentials: Jenkins credentials ID for SMTP user
        *   Default Recipients: `$DEFAULT_RECIPIENTS` (committers) or `team@example.com`
        *   Advanced: Enable "Use SSL" / "Use TLS"
    2.  **Test Configuration**: Click "Test configuration" button.
*   **Job Configuration**:
    *   *Post-build Actions* → "Editable Email Notification":
        *   *Project Recipient List*: `dev-team@example.com`
        *   *Triggers*: Add "Always", "Failure", "Success", etc.
        *   *Content Type*: HTML
*   **Pipeline**:
    ```groovy
    post {
        success {
            emailext (
                to: 'team@example.com',
                subject: 'Build Succeeded: ${JOB_NAME} [${BUILD_NUMBER}]',
                body: 'Check console output at ${BUILD_URL}'
            )
        }
        failure {
            emailext (to: 'team@example.com', subject: 'FAILED: ${JOB_NAME}', body: '${BUILD_LOG}')
        }
    }
    ```

#### **2. Extended Email Plugin (Custom Templates)**
*   **Why**: Customize email content beyond defaults (include logs, trends, links).
*   **Setup**:
    1.  Install "Email Extension Plugin".
    2.  *Manage Jenkins → Configure System* → **Extended E-mail Notification**:
        *   *Default Content*: Use Jelly/Groovy templates (see `$JENKINS_HOME/email-templates`).
        *   *Default Subject*: `[Jenkins] ${BUILD_STATUS}: ${JOB_NAME} [${BUILD_NUMBER}]`
*   **Custom Template Example** (`html.template`):
    ```html
    <h1>Build ${BUILD_STATUS}: ${JOB_NAME} #${BUILD_NUMBER}</h1>
    <p>Commit: ${GIT_COMMIT} by ${GIT_AUTHOR}</p>
    <p><a href="${BUILD_URL}">View Console Output</a></p>
    <h2>Test Results</h2>
    ${TEST_COUNTS,var="total"} tests total, ${TEST_COUNTS,var="fail"} failed.
    <img src="${BUILD_URL}/test/trend" />
    ```
*   **Use in Job**:
    *   *Editable Email Notification* → *Default Content* → `${SCRIPT, template="html.template"}`
*   **Variables**: Use `${BUILD_URL}`, `${GIT_BRANCH}`, `${TEST_COUNTS}`, `${BUILD_LOG,maxLines=20}`.

#### **3. Slack, Microsoft Teams, Discord Integration**
*   **Slack**:
    1.  Create Slack App → Add "Incoming Webhooks" → Copy Webhook URL.
    2.  Install "Slack Notification Plugin".
    3.  *Manage Jenkins → Configure System* → **Slack**:
        *   Workspace: `yourteam.slack.com`
        *   Credential: "Secret text" with Webhook URL
        *   Channel: `#jenkins` (or `@user`)
    4.  **Pipeline**:
        ```groovy
        post {
            success {
                slackSend channel: '#jenkins', 
                          color: 'good', 
                          message: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            }
            failure {
                slackSend (channel: '#alerts', 
                           message: "FAILED: <${env.BUILD_URL}|${env.JOB_NAME}>",
                           attachments: [[text: "${currentBuild.log(20)}"]])
            }
        }
        ```
*   **Microsoft Teams**:
    1.  Create Incoming Webhook in Teams channel → Copy URL.
    2.  Install "Microsoft Teams Notifier Plugin".
    3.  **Pipeline**:
        ```groovy
        teamsSend (color: '#33cc33', 
                   message: "Build succeeded: ${env.JOB_NAME}", 
                   webhookUrl: 'YOUR_WEBHOOK_URL')
        ```
*   **Discord**:
    1.  Create Webhook in Discord channel → Copy URL.
    2.  Use "Discord Notifier Plugin" or generic webhook:
        ```groovy
        httpRequest consoleLogResponseBody: true,
                    contentType: 'APPLICATION_JSON',
                    httpMode: 'POST',
                    requestBody: '{"content": "Build ${currentBuild.result}"}',
                    url: 'DISCORD_WEBHOOK_URL'
        ```

#### **4. Notifying on Success/Failure/Unstable**
*   **Triggers**:
    *   **Always**: Every build.
    *   **Success**: Only when build passes.
    *   **Failure**: Only when build fails (compilation, tests).
    *   **Unstable**: Tests passed but coverage/security thresholds not met.
    *   **Fixed**: Build was unstable/failing, now successful.
    *   **Still Failing**: Consecutive failures.
*   **Configuration**:
    *   *Editable Email Notification* → *Add Trigger* → Select condition.
    *   **Pipeline Example**:
        ```groovy
        post {
            success { /* ... */ }
            unstable { 
                emailext (to: 'qa-team@example.com', 
                          subject: 'UNSTABLE: ${JOB_NAME}', 
                          body: 'Coverage below threshold!') 
            }
            failure { /* ... */ }
            fixed { 
                slackSend (message: "Fixed: ${JOB_NAME} is GREEN!", color: 'good') 
            }
        }
        ```
*   **Best Practice**: Notify different channels per status (e.g., Slack #alerts for failures, #builds for successes).

#### **5. Build Monitor View Plugin**
*   **What it is**: Full-screen dashboard showing real-time build status (for team TVs).
*   **Setup**:
    1.  Install "Build Monitor View" plugin.
    2.  *New View* → "Build Monitor View" → Name it (e.g., "Team Dashboard").
    3.  Configure:
        *   *Jobs*: Select jobs to display (use regex like `project-*`).
        *   *Columns*: Number of jobs per row.
        *   *Refresh Interval*: `10` seconds.
        *   *Job Filters*: Show only running or failing jobs.
*   **Usage**:
    *   Access via `JENKINS_URL/view/Team%20Dashboard/`
    *   **Keyboard Shortcuts**: `F` (Fullscreen), `R` (Refresh), `C` (Collapse)
*   **Customization**:
    *   *CSS*: Override styles via `Manage Jenkins → Script Console` (advanced).
    *   *Icons*: Use custom icons for job statuses.

#### **6. Dashboard Views and Custom Reports**
*   **Dashboard Views Plugin**:
    *   Create customizable dashboards with widgets (test results, coverage, SCM changes).
    *   *New View* → "Dashboard View" → Add widgets (e.g., "JUnit Test Result Trend", "Build Statistics").
*   **Embedding Reports**:
    *   **Test Results**: Use `junit` trend graph URL: `JENKINS_URL/job/MyJob/test/trend`
    *   **Coverage**: `JENKINS_URL/job/MyJob/jacoco/api/json`
    *   **SonarQube**: Embed SonarQube project dashboard via iframe.
*   **Custom HTML Reports**:
    1.  Generate report (e.g., `mvn site` creates `target/site/index.html`).
    2.  Archive artifact: `archiveArtifacts 'target/site/**'`
    3.  Access via `JENKINS_URL/job/MyJob/lastSuccessfulBuild/artifact/target/site/index.html`
*   **Advanced Reporting**:
    *   **Plot Plugin**: Graph custom metrics (e.g., `sh 'echo "METRIC=$VALUE" > build.properties'` → Plot `build.properties`).
    *   **HTML Publisher Plugin**: Publish full HTML reports (e.g., Selenium, JMeter).
    *   **Custom Scripted Dashboard**: Use Jenkins Script Console to generate JSON reports.

---

### **Critical Cross-Cutting Best Practices**
1.  **Security First**:
    *   Store credentials in Jenkins Credentials Manager (never in scripts).
    *   Restrict plugin installations to trusted sources.
    *   Use agent-to-master security (JNLP secrets, SSH).
2.  **Idempotency**: Ensure builds are repeatable (clean workspace, immutable dependencies).
3.  **Pipeline as Code**: Always use `Jenkinsfile` (Declarative Pipeline) over UI configuration.
4.  **Modularize**: Break pipelines into shared libraries (e.g., `@Library('my-lib')`).
5.  **Monitor Jenkins**: Track disk space, queue length, plugin updates.
6.  **Disaster Recovery**: Backup `JENKINS_HOME` regularly (includes jobs, credentials, plugins).

---
