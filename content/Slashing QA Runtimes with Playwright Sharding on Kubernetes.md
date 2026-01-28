---
title: Slashing QA Runtimes with Playwright Sharding on Kubernetes
tags:
  - DevOps
  - Kubernetes
  - Playwright
  - Testing
  - CI/CD
---

**By Niranchan D, DevOps Team**

If you’re in DevOps, you know the feeling. You get a notification that the nightly QA regression suite has started. You glance at the Jenkins build, see it’s chugging along on a beefy worker node, and you know you won’t see the results for hours. It’s a monolithic beast, a single point of failure. A single instance failure wipes out the complete execution, and you’re back to “ZERO”.

This was our reality for a long time. While they were powerful tools, our traditional Pytest and Selenium suites, with their hundreds of test cases, presented a classic infrastructure challenge. Running them as a single, monolithic task meant they were slow to complete and fragile; any interruption meant starting over.

Then, a beacon appeared. Our QA team started adopting Playwright for their testing, and a particular feature got our DevOps Spidey-senses tingling: **sharding**.

### The Old Way: QA on One Fragile Node

Let’s quickly paint the “before” picture.

![[jenkins-pipeline-bottleneck.png]]

> **Traditional Jenkins Pipeline:** Sequential bottleneck with a single large worker instance running tests one after another, leading to slow execution and resource waste.
> **Duration:** Hours. Sometimes, many hours.
> **The Risk:** If anything went wrong with the instance — a random crash, a resource spike, a network glitch — the whole multi-hour process was toast. We’d have to restart the entire thing.

We weren’t just wasting time; we were creating a massive bottleneck for our release cycle. We needed a more resilient, more parallel, and frankly, a more modern way to do this.

### The “Aha!” Moment: Sharding + Kubernetes

Playwright’s `--shard` flag is simple but brilliant. It lets you split your test suite into a number of independent “shards” or chunks. For example, you can tell Playwright, “Run this suite, but only test case chunk 1 out of 5” (`--shard=1/5`), and another runner can execute chunk 2 out of 5, and so on.

As soon as we saw this, the solution became obvious. Instead of one giant job, why not many small, independent jobs? And what’s the perfect tool for running many small, independent jobs?

**Kubernetes Jobs.**

By combining Playwright’s application-level sharding with Kubernetes’ infrastructure-level job management, we could build a system that was faster, cheaper, and way more robust.

### The Solution: One K8s Job Per Shard

Here’s the master plan we cooked up:

![[k8s-playwright-architecture.png]]

1.  A Jenkins pipeline now triggers a **Kubernetes Job** instead of running tests directly.
2.  We use an **Indexed Job** in Kubernetes. This is the secret sauce. It allows us to run multiple pods in parallel for the job, and each pod gets a unique index number (0, 1, 2, etc.) which we use to identify and process the data later.

> **Why Indexed Jobs?**
> Kubernetes Indexed Jobs provide a structured way to run parallel tasks with unique identifiers, making them ideal for Playwright sharding where each pod needs a distinct chunk of tests. Unlike standard Jobs, the `completionMode: Indexed` assigns each pod a unique index (e.g., 0 to N-1) via the `JOB_COMPLETION_INDEX` environment variable, which we map directly to Playwright's `--shard` flag for seamless test distribution. This ensures balanced workloads without custom scripting, as Kubernetes tracks completions per index—preventing overlaps and enabling the Job to complete only when all indices succeed.

3.  Each pod runs the same container image but is responsible for executing just one shard of the test suite, using its unique index to figure out which shard.
4.  To ensure we don’t lose progress, each pod, upon finishing its shard, uploads its test results (Allure reports, `.last-run.json`, etc.) to a dedicated folder in an S3 bucket.
5.  A final “aggregator” step in our pipeline waits for all jobs to complete, then pulls all the results from S3 to generate a single, unified report.
6.  If one pod fails, Kubernetes’ `backoffLimit` will try to rerun it a couple of times. But crucially, the other 19 (or however many) successful shards are already done, their results safely stored in S3. No more starting from scratch!

### Let’s Get Technical: A Peek at the K8s Job

Here’s a simplified look at the Kubernetes Job manifest. It’s the heart of our solution.

```groovy
pipeline {
 agent any parameters {
 string(name: 'SHARDS', defaultValue: '3', description: 'Number of shards')
 }
 stages {
 stage('Apply Kubernetes Job') {
 steps {
 script {
 // Read the bash script. our QA maintains this file and we just review upon changes.
 def scriptContent = readFile 'test-script.sh'
 // Dynamically replace placeholders in the script content.
 scriptContent = scriptContent.replace('${SHARDS}', params.SHARDS)
 scriptContent = scriptContent.replace('${BUILD_NUMBER}', env.BUILD_NUMBER)
 def jobYaml = """
 apiVersion: batch/v1
 kind: Job
 metadata:
 name: playwright-test-shard-${BUILD_NUMBER}
 spec:
 parallelism: ${params.SHARDS} # Launches all shards in parallel
 completions: ${params.SHARDS} # Requires all shards to succeed for Job completion
 completionMode: Indexed
 template:
 spec:
 serviceAccountName: s3-access-qa
 containers:
 - name: playwright
 image: ${params.ECR_IMAGE_URI} // Dynamically passed from params
 command:
 - "sh"
 - "-c"
 - |
 ${scriptContent} // Injected bash script from file
 env:
 - name: JOB_COMPLETION_INDEX
 valueFrom:
 fieldRef:
 fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
 restartPolicy: Never
 nodeSelector:
 application: regression
 backoffLimit: 2
 """
 writeFile file: 'job.yaml', text: jobYaml
 sh 'kubectl apply -f job.yaml'
 }
 }
 }
```

And a bash script that runs inside each indexed job:

```bash
# sample test-script.sh
SHARD_NUMBER=$((JOB_COMPLETION_INDEX + 1))
echo "--- Starting Test Run: Shard ${SHARD_NUMBER}/${SHARDS} ---"
npm run regression -- --shard=\${SHARD_NUMBER}/${SHARDS}
echo "--- Test Run Finished. Uploading results to S3... ---"
aws s3 cp my-allure-results "s3://our-qa-bucket/results/${BUILD_NUMBER}/job_\${JOB_COMPLETION_INDEX}/my-allure-results" --recursive
aws s3 cp test-results "s3://our-qa-bucket/results/${BUILD_NUMBER}/job_\${JOB_COMPLETION_INDEX}/test-results" --recursive
echo "--- Upload Complete: Shard ${JOB_COMPLETION_INDEX} ---"
```

The beauty of this is its simplicity and power.
*   `completionMode: Indexed` and the `JOB_COMPLETION_INDEX` environment variable are doing the heavy lifting of assigning work.
*   The command is just a simple shell script that dynamically figures out its shard number and tells Playwright what to do.
*   The `aws s3 cp` command ensures each piece of the puzzle is saved independently.

### The Wins: Speed, Resilience, and Sanity

So, what did we gain?

*   **Massive Speed Improvement:** Running 20 shards in parallel is, unsurprisingly, a lot faster than running one monolithic suite. Our hours-long runs are now down to minutes.
*   **True Resilience:** A flaky test or a pod failure now only affects one shard. The other 19 complete successfully. We only need to investigate or re-run that one small piece.
*   **Efficiency:** We leverage our existing Kubernetes cluster, spinning up pods only when needed. No more expensive idle time.
*   **Scalability:** Got more tests? We just bump the `${params.SHARDS}` value. The system scales effortlessly.

### What’s Next

We’re already planning improvements:

*   **Dynamic sharding:** Auto-adjust the number of shards at runtime based on the suite size.
*   **Dynamic Provisioning:** Scale Kubernetes node capacity up or down to match the shard count and test-suite configuration, ensuring you never over- or under-allocate resources.
*   **Intelligent retry:** Layer Playwright’s per-test retries setting with Kubernetes `backoffLimit` to handle both flaky tests and transient pod crashes.
*   **Cost optimization:** Spot instances for non-critical runs.
*   **Multi-environment:** Run the same shards in parallel across multiple target environments.

Stop orchestrating monoliths; start orchestrating outcomes.

*Originally published on Medium.*
