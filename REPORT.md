# SOFE3980U Lab 3 Part 2 Report
## Continuous Integration and Deployment with Jenkins on GKE

**Name:** Emmanuel Omole  
**Student number:** 101004432  
**Date:** 2026-09-27

**GitHub repository:** https://github.com/Omoleen/SOFE3980U-Lab3-Part2

---

## 1. Setup

| Item | Value |
| --- | --- |
| GKE cluster | `sofe3980u-cluster`, zone `northamerica-northeast2-a`, 2 x `e2-standard-4` |
| Jenkins | Helm chart `jenkinsci/jenkins` 5.9.64, Jenkins 2.568.3, release `cd-jenkins` |
| Artifact Registry | Docker repository `sofe3980u` in `northamerica-northeast2` |
| Service account | `jenkins-sa`, JSON key stored in Jenkins as the secret file `service_account` |
| Jenkins credentials | `GitHub_token`, `service_account`, `project_id`, `repo_path`, `cluster_name`, `cluster_zone` |
| Jobs | `binaryCalculate_mvn` (Maven project), `BinaryCalculator_pipeline` (`Jenkinsfile`), `BinaryCalculator_cicd` (`Jenkinsfile_v2`) |
| Jenkins URL | `http://34.130.132.99:8080` |
| Webhook | `http://34.130.132.99:8080/github-webhook/`, content type JSON, push events |

Jenkins was installed with the lab `values.yaml`, which exposes the controller
through a `LoadBalancer` service on port 8080. All three jobs use the trigger
*GitHub hook trigger for GITScm polling*, so a single webhook starts every job
on each push.

Two things differed from the handout:

- **CSRF proxy compatibility.** The *Enable proxy compatibility* checkbox no
  longer exists in Jenkins 2.568. Crumbs are now tied to the web session rather
  than to the client IP, so the connection issue it fixed does not occur and the
  step was skipped.
- **Agent memory.** The controller has no executors, so every build runs in an
  agent pod created by the Kubernetes plugin. With the lab's 512 MiB limit the
  first `binaryCalculate_mvn` run lost its agent channel partway through the
  build (`ClosedChannelException`). A Maven project job starts a second JVM for
  Maven next to the agent JVM, which does not fit in 512 MiB. Raising the agent
  limit to 2 GiB fixed it. The two pipeline jobs, which run `mvn` from a plain
  `sh` step, passed at the original limit.

## 2. Continuous Integration

### 2.1 Maven project job

`binaryCalculate_mvn` clones the repository, runs `clean package` against
`BinaryCalculatorWebapp/pom.xml`, and then runs *Set GitHub commit status
(universal)*. That post-build action uses the `GitHub_token` credential to
write the build result back to the commit, which is what puts the green check
and the link to the Jenkins build next to the commit on GitHub.

### 2.2 Pipeline job

`BinaryCalculator_pipeline` reads its steps from
`BinaryCalculatorWebapp/Jenkinsfile` on `*/main`. The file has four stages:
`Init` prints a message and lists the workspace, `test` runs
`mvn clean test`, `build` runs `mvn package -DskipTests`, and `Deploy` is a
placeholder that prints a string. The difference from the Maven job is that
the build process is kept in the repository and versioned with the code, rather
than stored as job configuration in Jenkins.

## 3. Continuous Deployment

`BinaryCalculator_cicd` runs `Jenkinsfile_v2`. The `environment` block loads the
five credentials into variables, and the `agent` block defines a pod with an
extra `gcloud` container from `google/cloud-sdk:latest`. After `test` and
`build`, three stages run inside that container:

1. **containerize** authenticates as `jenkins-sa`, then runs
   `gcloud builds submit`, which uploads the directory to Cloud Build, builds
   the image from the `Dockerfile` and pushes it to Artifact Registry.
2. **deployment** fetches cluster credentials, deletes
   `binarycalculator-deployment` if it exists, and recreates it from the new
   image.
3. **service** exposes the deployment through the `LoadBalancer` service
   `binarycalculator-service` (only created once) and prints its external IP.

Cloud Build pushes with the Compute Engine default service account, which is
why that account needs the Artifact Registry Writer role.

## 4. Discussion

**What do pipeline, node, agent, stage and steps mean in the context of
Jenkins?**

- **Pipeline:** the whole build and delivery process written as code, usually in
  a `Jenkinsfile` stored in the repository. In declarative syntax it is the
  top-level `pipeline { }` block that holds the agent, environment, tools and
  stages. Because it lives in source control, a change to the process is
  reviewed and versioned like any other code change.
- **Node:** any machine that can run Jenkins work, either the controller or an
  agent. A node offers executors (slots that run one build each) and a
  workspace directory. In scripted pipelines `node { }` is the step that grabs an
  executor and a workspace. In this lab the controller has zero executors, so
  every node is a short-lived Kubernetes pod.
- **Agent:** in declarative syntax, the directive that tells Jenkins where the
  pipeline or a stage runs. `agent any` in `Jenkinsfile` accepts any node, which
  here is the chart's default pod template. `agent { kubernetes { yaml ... } }` in
  `Jenkinsfile_v2` describes the pod itself, adding the `gcloud` container that
  `container('gcloud')` steps run in. The word also names the worker process
  that connects to the controller, here through the tunnel
  `cd-jenkins-agent:50000`.
- **Stage:** a named group of steps that is one logical phase of the pipeline,
  such as `test`, `build`, `containerize`, `deployment` and `service`. Stages
  run in order, show up as columns in the stage view with their own duration
  and result, and a failing stage stops the rest of the pipeline.
- **Steps:** the individual actions inside a stage, and the smallest unit of
  work Jenkins runs. Examples from the lab files are `sh`, `echo`, `dir`,
  `container`, `script` and `tool`.

In short: a pipeline is made of stages, each stage is a list of steps, and the
agent directive decides which node those steps run on.

## 5. Design

### 5.1 Changes

The webapp was brought up to the Lab 1 version of the calculator.

| File | Change |
| --- | --- |
| `Binary.java` | Replaced with the Lab 1 class: `or`, `and` and `multiply` (shift-and-add), plus null and empty input handling in the constructor |
| `BinaryController.java` | The `*`, `\|` and `&` operators now return a result page. Unknown operators return the `error` view. The original code returned `"Error"`, which does not match `error.html` on a case-sensitive file system |
| `BinaryAPIController.java` | Added `/multiply`, `/or` and `/and`, each with a `_json` variant that returns a `BinaryAPIResult` |
| `BinaryTest.java` | Added the 27 Lab 1 unit tests for `Binary` |
| `BinaryControllerTest.java` | Added 4 tests: POST with `*`, `\|` and `&`, and an invalid operator |
| `BinaryAPIControllerTest.java` | Added 6 tests: plain and JSON forms of each new endpoint |
| `.gitignore` | Excludes `target/` |

The calculator page already had buttons for `*`, `|` and `&`, so no template
changes were needed. Before this change those buttons fell through to the
controller's default branch.

### 5.2 Test results

The suite went from 13 tests to 50:

```
Tests run: 27 - BinaryTest
Tests run:  6 - HelloAPIControllerTest
Tests run:  8 - BinaryAPIControllerTest
Tests run:  7 - BinaryControllerTest
Tests run:  2 - HelloControllerTest
Tests run: 50, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

Expected values were checked in decimal. For example, `/multiply` with `111`
and `1010` asserts `1000110` (7 x 10 = 70), and `/and` with the same operands
asserts `10` (7 & 10 = 2).

### 5.3 Deployment

Pushing the change to `main` started all three jobs through the webhook. The
Maven job posted its status to the commit, the pipeline job ran its four
stages, and `BinaryCalculator_cicd` rebuilt the image and replaced the
deployment. The last line of its console output is the external IP of
`binarycalculator-service`, and the calculator is served at
`http://34.130.18.211:8080`, where `*`, `|` and `&` now return results.
