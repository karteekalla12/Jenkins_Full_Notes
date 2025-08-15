### **Part 9: Monitoring, Logging & Performance**

---

## **Chapter 26: Monitoring Jenkins**

### **1. Built-in System Information**
*   **What it is:** Jenkins exposes a wealth of internal system data via HTTP endpoints and the web UI.
*   **Key Endpoints & Access:**
    *   **`/systemInfo` (Web UI):** Navigate to `Jenkins URL -> Manage Jenkins -> System Information`.  
        *   **Critical Data Included:**
            *   **JVM Details:** Java version, vendor, arguments (`-Xmx`, `-XX`), heap usage, GC stats.
            *   **OS Details:** OS name, version, architecture, available processors, total/free memory.
            *   **File System:** Jenkins home path, disk space usage (`/` and Jenkins home partitions).
            *   **Environment Variables:** All env vars visible to the Jenkins master process.
            *   **System Properties:** Java system properties (e.g., `user.dir`, `java.io.tmpdir`).
            *   **Jenkins Configuration:** Version, active plugins (name/version), security settings, agent protocols.
            *   **Thread Dumps:** Link to `/threadDump` (shows stack traces of all running threads - **vital for hangs**).
    *   **`/systemInfo` (Raw API):** `curl -u USER:TOKEN http://JENKINS_URL/systemInfo` (Returns JSON/XML).
    *   **`/whoAmI` (API):** `curl -u USER:TOKEN http://JENKINS_URL/whoAmI/api/json` - Confirms authenticated user/permissions.
*   **Why it Matters:** First stop for diagnosing performance bottlenecks, resource exhaustion (CPU/MEM/DISK), plugin conflicts, or unexpected environment changes. **Check this BEFORE installing plugins for basic health.**

### **2. Jenkins Monitoring Plugins (Prometheus, Grafana)**
*   **The Need:** Built-in info is static/snapshot. Plugins provide **continuous metrics collection, visualization, and alerting**.
*   **a) Prometheus Plugin (`prometheus`):**
    *   **Function:** Exposes Jenkins metrics in Prometheus format (`/prometheus` endpoint).
    *   **Key Metrics Collected:**
        *   **Build Metrics:** `jenkins_builds_duration_milliseconds_count`, `jenkins_builds_queue_duration_milliseconds`, `jenkins_builds_result{result="SUCCESS|FAILURE|..."}`
        *   **System Metrics:** `jenkins_jvm_memory_bytes_used`, `jenkins_jvm_threads`, `jenkins_system_cpu_load_average`, `jenkins_agents_count`
        *   **Plugin Metrics:** `jenkins_plugin_installations_count`, `jenkins_plugin_failures_count`
        *   **Queue Metrics:** `jenkins_queue_buildable_count`, `jenkins_queue_pending_count`
        *   **Executor Metrics:** `jenkins_executor_idle_count`, `jenkins_executor_busy_count`
    *   **Setup:**
        1.  Install `Prometheus Metrics` plugin (Manage Jenkins -> Plugins).
        2.  Configure: `Manage Jenkins -> Configure System -> Prometheus`: Enable endpoint, set port (default `/prometheus`), configure security (ACLs).
        3.  Add Jenkins target to Prometheus `scrape_configs`:
            ```yaml
            scrape_configs:
              - job_name: 'jenkins'
                metrics_path: '/prometheus'
                static_configs:
                  - targets: ['jenkins-server:8080']
                basic_auth:  # If secured
                  username: 'monitoring-user'
                  password: 'api-token'
            ```
    *   **Critical Note:** Requires proper authentication setup (API token/user with `ViewStatus` permission).
*   **b) Grafana Integration:**
    *   **Function:** Visualize Prometheus (or other data source) metrics in dashboards.
    *   **Setup:**
        1.  Install Grafana.
        2.  Add Prometheus as a Data Source in Grafana.
        3.  Import/Build Dashboards:
            *   **Official Dashboards:** Jenkins community provides Grafana JSON templates (search Grafana Dashboards site for "Jenkins").
            *   **Key Dashboard Areas:**
                *   **Master Health:** JVM Heap/Memory, GC Pauses, CPU Load, Thread Count.
                *   **Build Performance:** Avg Build Duration (by job), Queue Time, Failure Rates (trends).
                *   **Agent Utilization:** Busy/Idle Executors (per agent/label), Agent Offline Events.
                *   **System Resources:** Disk I/O (if monitored via Node Exporter), Network.
    *   **Why Grafana?** Enables trend analysis (e.g., "Build times increased 200% after plugin X update"), setting alerts (e.g., "Alert if queue time > 5 mins for 10 mins"), and correlating Jenkins metrics with infrastructure metrics.

### **3. Tracking Build Metrics and Trends**
*   **Beyond Snapshots:** Monitoring *changes* over time is crucial for proactive management.
*   **Methods & Tools:**
    *   **Prometheus + Grafana (Primary Method):** As above. Track:
        *   `rate(jenkins_builds_duration_milliseconds_sum[1h]) / rate(jenkins_builds_duration_milliseconds_count[1h])` (Avg Duration)
        *   `increase(jenkins_builds_result{result="FAILURE"}[7d])` (Failure Count Trend)
        *   `jenkins_builds_queue_duration_milliseconds{quantile="0.95"}` (95th Percentile Queue Time)
    *   **Jenkins Plugins:**
        *   **`build-time-blame` Plugin:** Identifies slow stages/steps *within* pipelines.
        *   **`build-trend-report` Plugin:** Generates historical trend reports (CSV/HTML) for builds.
        *   **`analysis-model` (Used by Warnings Next Gen):** Tracks code quality trends (bugs, warnings) over builds.
    *   **Custom Scripting:** Use Jenkins Script Console or `curl` + `jq` to extract historical data from Jenkins API (`/job/JOBNANE/api/json?tree=builds[number,timestamp,duration,result]`) and feed into external DB (e.g., InfluxDB).
*   **Key Trends to Monitor:**
    *   **Gradual Increase in Build Duration:** Indicates code bloat, inefficient tests, or infrastructure degradation.
    *   **Spike in Queue Time:** Signals insufficient executors or resource contention.
    *   **Rising Failure Rate:** Points to flaky tests, unstable dependencies, or recent bad deployments.
    *   **Increasing GC Pauses:** Warning sign of memory pressure or heap misconfiguration.

### **4. Health Check Endpoints**
*   **Purpose:** Provide machine-readable status for load balancers, orchestrators (K8s), or monitoring systems to determine if Jenkins is "ready" or "live".
*   **Critical Endpoints:**
    *   **`/monitoring` (Requires `monitoring` plugin):**
        *   **Function:** Aggregates health checks from various plugins (disk space, thread pool, plugin updates).
        *   **Response:** `HTTP 200 OK` if all checks pass. `HTTP 503 Service Unavailable` with JSON detailing failing checks if any fail.
        *   **Example Failing Response:**
            ```json
            {
              "status": "error",
              "checks": [
                {
                  "name": "Disk Space Monitoring",
                  "status": "error",
                  "message": "Free space on /var/jenkins_home is below 10% (5.2 GB)"
                }
              ]
            }
            ```
    *   **`/systemInfo` (Basic Readiness):** `HTTP 200` indicates Jenkins is up and responding, but **NOT** necessarily healthy (e.g., DB connection might be dead).
    *   **`/login` (Simple Liveness):** `HTTP 200` usually indicates the web container is alive. Less reliable than `/monitoring`.
    *   **`/api/json` (API Readiness):** `HTTP 200` confirms the core API is functional.
*   **Kubernetes Integration (Critical for Cloud):**
    *   **Liveness Probe:** `/login` (Simple check Jenkins process is responsive).
    *   **Readiness Probe:** `/monitoring` (Ensures Jenkins is *actually ready* to serve traffic - no disk full, thread starvation, etc.). **This is essential to prevent routing traffic to a broken instance.**

### **5. Performance Monitoring of Master and Agents**
*   **Master Monitoring (The Critical Bottleneck):**
    *   **Key Metrics:**
        *   **JVM Heap:** `used / max` (Target <70-80% consistently; spikes ok). Watch for frequent/full GCs (`GC pause time`).
        *   **CPU Load:** Sustained load > `(available CPUs) * 0.7` indicates overload.
        *   **Threads:** `jenkins_jvm_threads` (Too high = potential deadlock/thread leak). Check `/threadDump`.
        *   **Disk I/O:** High latency on Jenkins home (`/var/jenkins_home`) or `JENKINS_HOME/caches` cripples performance. Monitor `iowait`.
        *   **Network:** Bandwidth saturation if many agents/repos.
    *   **Tools:** Prometheus/Grafana (JVM metrics), OS tools (`top`, `htop`, `vmstat`, `iostat`, `netstat`), APM tools (New Relic, Datadog).
*   **Agent Monitoring (Where the Work Happens):**
    *   **Key Metrics (Per Agent/Label):**
        *   **Executor State:** `Busy` vs `Idle` count (Grafana: `jenkins_executor_busy_count`).
        *   **Agent Status:** `Online` vs `Offline` (Track disconnects: `jenkins_agents_offline_count`).
        *   **Agent Resource Usage:** CPU, MEM, DISK on the agent host (Requires host-level monitoring like Node Exporter).
        *   **Build Duration on Agent:** Compare to master queue time to isolate agent slowness.
    *   **Why Monitor Agents?** Slow agents bottleneck the whole system. Frequent disconnects indicate network issues or agent instability. Resource exhaustion on agents causes build failures.
    *   **Tools:** Prometheus (Jenkins plugin metrics), Host-level monitoring (Node Exporter, CloudWatch), Jenkins UI (`Manage Jenkins -> Nodes`).

---

## **Chapter 27: Logging and Troubleshooting**

### **1. Accessing Jenkins Logs (`jenkins.log`)**
*   **Location:**
    *   **Default (Standalone):** `$JENKINS_HOME/jenkins.log` (e.g., `/var/lib/jenkins/jenkins.log` on Linux, `C:\Program Files\Jenkins\jenkins.log` on Windows).
    *   **Systemd (Linux):** `journalctl -u jenkins` (Often configured to redirect `stdout/stderr` here).
    *   **Docker:** `docker logs <jenkins-container>` or volume-mounted log file.
    *   **Kubernetes:** `kubectl logs <jenkins-pod>`.
*   **Log Levels & Configuration:**
    *   **Manage Jenkins -> System Log -> Log Recorders:** Create custom log recorders.
        *   **Name:** e.g., "Troubleshooting".
        *   **Loggers:**
            *   `jenkins` (Core Jenkins)
            *   `hudson` (Legacy core)
            *   `org.jenkinsci` (Plugins)
            *   `com.cloudbees.jenkins.plugins.ssh` (Example plugin logger)
        *   **Log Level:** `ALL`, `FINEST`, `FINE`, `INFO`, `WARNING`, `SEVERE`. **Use `FINE` or `FINER` for deep debugging.**
    *   **`/script` Console:** Dynamically change log levels:
        ```groovy
        import java.util.logging.*
        Logger.getLogger("hudson.model.Run").setLevel(Level.FINE)
        Logger.getLogger("org.jenkinsci.plugins.workflow.job.WorkflowRun").setLevel(Level.FINER)
        ```
*   **Critical Log Areas:**
    *   **Startup:** Look for plugin load errors, config validation failures, port conflicts.
    *   **Security:** Authentication/authorization failures (`WARNING: Failed to authenticate user...`).
    *   **Agent Connect:** `Agent successfully connected` / `Terminating ... due to exception: ...`.
    *   **Build Execution:** SCM checkout errors, step execution logs, process termination.
    *   **GC Events:** `Full GC` pauses (indicate memory pressure).
    *   **Thread Dumps:** Triggered automatically on deadlock detection or manually via `/threadDump`.

### **2. Diagnosing Build Failures**
*   **Systematic Approach:**
    1.  **Check Build Console Output:** *Always start here.* Look for the **first error** (often masked by later stack traces).
    2.  **Identify Failure Stage:** SCM Checkout? Build Tool (Maven/Gradle)? Test Execution? Post-Build Step?
    3.  **Check Relevant Logs:**
        *   **Build Log:** Full console output (includes all shell steps).
        *   **`jenkins.log`:** Look for exceptions *around* the build timestamp. Search for the build number (`#123`).
        *   **Agent Logs:** If agent-specific (e.g., `agent.log` on the agent machine).
    4.  **Reproduce Locally:** Can you run the failing command *exactly* on the agent OS? (Crucial for env issues).
    5.  **Check Dependencies:** Did a library version change? Is a remote service (DB, API) down?
    6.  **Check Resource Limits:** OOM Killer? Disk full on agent? (`df -h`, `dmesg | grep -i kill`).
*   **Common Failure Patterns:**
    *   **SCM Failure:** `ERROR: Error cloning remote repo...` -> Check repo URL, credentials, network, proxy.
    *   **Build Tool Failure:** `mvn: command not found` -> Check PATH on agent, tool config in Jenkins.
    *   **Test Failure:** `Tests run: 10,  Failures: 1` -> Examine test report/output.
    *   **Resource Exhaustion:** `java.lang.OutOfMemoryError`, `java.io.IOException: No space left on device`.
    *   **Permission Denied:** `hudson.AbortException: java.io.IOException: Permission denied` -> Check file/dir ownership on agent.

### **3. Debugging Pipelines (using `sh 'set -x'`, `echo`)**
*   **Core Principle:** Make the invisible visible. Inject verbosity into your pipeline steps.
*   **Declarative Pipeline Debugging:**
    ```groovy
    pipeline {
        agent any
        stages {
            stage('Debug Me') {
                steps {
                    // 1. Shell Tracing (Most Powerful)
                    sh '''
                        set -x  # Enable command tracing (prints each command before execution)
                        echo "Current Dir: $(pwd)"
                        env       # Dump environment variables
                        ls -la    # List files
                        # ... your actual commands ...
                        set +x  # Disable tracing (optional, keeps log cleaner)
                    '''

                    // 2. Explicit Echo Statements
                    echo "BUILD_NUMBER: ${env.BUILD_NUMBER}"
                    echo "WORKSPACE: ${env.WORKSPACE}"
                    script {
                        // 3. Groovy Debugging
                        println "Java Home: ${tool 'JDK 17'}" // Print resolved tool path
                        def files = findFiles glob: '**/*.jar'
                        echo "Found JARs: ${files}"
                    }
                }
            }
        }
    }
    ```
*   **Scripted Pipeline Debugging:**
    ```groovy
    node {
    // Same shell tracing techniques within `sh` steps
    sh 'set -x && mvn clean install && set +x'

    // Groovy Debugging
    echo "Params: ${params}"
    println "Current Build: ${currentBuild}"
    currentBuild.rawBuild.getExecutor().getOwner().getRemote() // Get agent name

    // Pause for inspection (Dangerous in prod!)
    input "Check agent state? Agent: ${env.NODE_NAME}"
    }
    ```
*   **Advanced Techniques:**
    *   **`@NonCPS` for Complex Debugging:** Break out of CPS transformation for deep Groovy debugging (use sparingly).
    *   **`catchError` + `currentBuild.rawBuild`:** Capture detailed exception info within a stage.
    *   **`JENKINS_NODE_COOKIE` Trick:** On the agent, find the running process: `ps aux | grep $JENKINS_NODE_COOKIE`.

### **4. Common Errors and Fixes**
*   **`java.io.IOException: Connection reset` / `Agent went offline`**
    *   **Cause:** Network instability, agent JVM crash, master GC pause > agent timeout.
    *   **Fix:** Increase agent timeout (`Manage Jenkins -> System Configuration -> # of executors -> Advanced -> Agent -> Idle timeout`), check network/firewall, monitor master GC.
*   **`hudson.remoting.RequestAbortedException`**
    *   **Cause:** Usually agent disconnect *during* a build step.
    *   **Fix:** Same as above. Also check agent resource exhaustion (OOM).
*   **`No space left on device` (Even if `df` shows space)**
    *   **Cause:** Inode exhaustion (`df -i`), or Docker container disk quota hit.
    *   **Fix:** `df -i`, clean up small files (logs, temp files). For Docker, increase container disk size.
*   **`Failed to load plugins` / `Incompatible plugin`**
    *   **Cause:** Plugin dependency conflict, Jenkins core version mismatch.
    *   **Fix:** Check `jenkins.log` for exact error. Manually download compatible plugin `.hpi` files. Use `plugin-management` CLI tool.
*   **`403 No valid crumb was included in the request`**
    *   **Cause:** CSRF protection enabled, but API/script request lacks valid crumb.
    *   **Fix:** **DO NOT DISABLE CSRF!** Obtain crumb: `curl -u USER:TOKEN 'http://JENKINS_URL/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,":",//crumb)'`. Use it in subsequent requests: `-H "$CRUMB_HEADER"`.
*   **Pipeline Hangs Indefinitely**
    *   **Cause:** Deadlock, infinite loop, agent disconnect without termination signal, resource starvation.
    *   **Fix:** Check `/threadDump`, look for blocked threads. Check agent status. Use `timeout` step in pipelines. Monitor system resources.

### **5. Using Blue Ocean for Visualization**
*   **What it is:** Modern, user-friendly UI for Jenkins pipelines (requires `blueocean` plugin).
*   **Troubleshooting Advantages over Classic UI:**
    *   **Visual Pipeline Editor:** Drag-and-drop pipeline creation (less YAML/DSL errors).
    *   **Stage View Timeline:** Clear visualization of stage durations, parallel execution, and **bottlenecks** (longest bars). *Instantly see where time is spent.*
    *   **Step-Level Logs:** Click any stage/step to see *only* its logs. **No more scrolling through massive logs!**
    *   **Integrated Console Output:** Logs are displayed alongside the pipeline visualization.
    *   **Pull Request Integration:** Clear view of PR builds and status.
    *   **Pipeline-as-Code Navigation:** Easily jump from Blue Ocean view to the `Jenkinsfile` in SCM.
*   **How to Use for Debugging:**
    1.  Open a failed build in Blue Ocean.
    2.  **Identify the Failed Stage:** It will be highlighted in red.
    3.  **Click the Stage:** See its duration and status.
    4.  **Click the Specific Step** within the stage that failed (e.g., `sh`, `junit`).
    5.  **View Step Logs:** The right panel shows *only* the logs for that step. **This is the single biggest time-saver.**
    6.  **Compare Runs:** Use the timeline view to compare duration of the same stage across successful/failed builds.
*   **Limitation:** Primarily for Pipeline (Declarative/Scripted) jobs. Less useful for Freestyle jobs (though it shows them).

---
