# Jenkins Basics

> **TL;DR:** Jenkins is old but ubiquitous. Use **declarative pipelines** in a `Jenkinsfile`, run on **ephemeral agents** (K8s pods or Docker), and put shared logic into a **Shared Library**. Avoid free-style jobs in 2026.

## When You'll See It

- Enterprises with years of investment in Jenkins plugins.
- Air-gapped / regulated environments (banks, government, defense).
- Build flows talking to legacy systems (mainframe, on-prem artifact servers, vSphere).

Newer projects pick GitHub Actions / GitLab CI / Buildkite, but Jenkins knowledge is still expected.

## Architecture

- **Controller (formerly "master"):** UI + orchestrator. Stores config in `JENKINS_HOME` (back this up).
- **Agents (formerly "slaves"):** Where builds actually run. Ephemeral preferred.
- **Executors:** Slots per agent that can run a build.
- **Plugins:** Everything is a plugin. Update aggressively; old plugins are CVE-rich.

```
[Controller]  ---SSH/JNLP--->  [Static Agent VM]
              ---API--->        [K8s Pod Agent] (per build, ephemeral)
              ---Docker--->     [Docker Agent]  (per build, ephemeral)
```

## Declarative Pipeline (Jenkinsfile) — the modern way

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                    - name: node
                      image: node:20-alpine
                      command: ["sleep"]
                      args: ["infinity"]
                    - name: docker
                      image: docker:24
                      command: ["sleep"]
                      args: ["infinity"]
                      volumeMounts:
                        - { name: dind-sock, mountPath: /var/run }
                    - name: dind
                      image: docker:24-dind
                      securityContext: { privileged: true }
                      volumeMounts:
                        - { name: dind-sock, mountPath: /var/run }
                  volumes:
                    - name: dind-sock
                      emptyDir: {}
            '''
        }
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
    }

    environment {
        REGISTRY = 'ghcr.io/me'
        IMAGE    = "${REGISTRY}/api:${env.GIT_COMMIT.take(7)}"
        // Bind a Jenkins credential to env
        SLACK_WEBHOOK = credentials('slack-webhook')
    }

    parameters {
        choice(name: 'ENV', choices: ['staging', 'prod'], description: 'Target env')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false)
    }

    triggers {
        cron('H 6 * * 1-5')           // weekdays 6am-ish ("H" randomizes)
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Install') {
            steps {
                container('node') { sh 'npm ci' }
            }
        }

        stage('Lint & Test') {
            when { not { expression { params.SKIP_TESTS } } }
            parallel {
                stage('Lint')  { steps { container('node') { sh 'npm run lint' } } }
                stage('Test')  {
                    steps  { container('node') { sh 'npm test -- --ci' } }
                    post   { always { junit 'reports/junit.xml' } }
                }
            }
        }

        stage('Build image') {
            steps {
                container('docker') {
                    sh "docker build -t $IMAGE ."
                    withCredentials([usernamePassword(
                        credentialsId: 'ghcr-creds',
                        usernameVariable: 'U', passwordVariable: 'P')]) {
                        sh 'echo $P | docker login ghcr.io -u $U --password-stdin'
                        sh "docker push $IMAGE"
                    }
                }
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                input message: "Deploy ${IMAGE} to ${params.ENV}?", ok: 'Yes'
                container('node') {
                    sh "kubectl set image deploy/api api=${IMAGE} -n ${params.ENV}"
                    sh "kubectl rollout status deploy/api -n ${params.ENV}"
                }
            }
        }
    }

    post {
        success { echo 'OK' }
        failure {
            sh 'curl -sX POST -H "Content-Type: application/json" $SLACK_WEBHOOK \
                -d \'{"text":"Build failed"}\''
        }
        always {
            cleanWs()
        }
    }
}
```

### Declarative vs Scripted Pipeline

- **Declarative** (`pipeline { ... }`): structured, validated, restartable from any stage. Recommended.
- **Scripted** (`node { ... }`): raw Groovy. Powerful, but you own everything. Drop down into a `script {}` block inside declarative when you need it.

```groovy
script {
    def branches = ['a', 'b'].collectEntries { name ->
        [(name): { sh "./test ${name}" }]
    }
    parallel branches
}
```

## Agents — Make Them Ephemeral

### Static Agent (legacy)

A long-lived VM with Java + tools installed. Avoid: state accumulates, builds become non-reproducible.

### Docker Agent

```groovy
pipeline {
    agent { docker { image 'node:20-alpine' } }
    stages { stage('Test') { steps { sh 'npm test' } } }
}
```

Per-build fresh container — no state carries over.

### Kubernetes Agent (best for cloud)

The Pod template above. Each build = new Pod, dies after. Scales to thousands of concurrent builds, no agent maintenance.

## Credentials

In Jenkins UI: **Manage Jenkins → Credentials**. Reference by ID:

```groovy
withCredentials([
    string(credentialsId: 'npm-token', variable: 'NPM_TOKEN'),
    usernamePassword(credentialsId: 'aws',
        usernameVariable: 'AWS_ACCESS_KEY_ID',
        passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
    sshUserPrivateKey(credentialsId: 'deploy-key', keyFileVariable: 'KEY')
]) {
    sh 'npm publish'
    sh 'aws s3 sync ./dist s3://bucket'
    sh 'ssh -i $KEY user@host ./deploy.sh'
}
```

Jenkins masks these in logs (if you echo them, they get `****`). Don't trust it as your only defense.

## Shared Libraries — DRY Across Jenkinsfiles

Move common logic (build, push, deploy) into a Git repo, then import:

```
my-shared-lib/
└── vars/
    ├── deployToK8s.groovy
    ├── notifySlack.groovy
    └── buildAndPush.groovy
```

```groovy
// vars/deployToK8s.groovy
def call(Map cfg) {
    sh """
        kubectl set image deploy/${cfg.deploy} \
            ${cfg.container}=${cfg.image} -n ${cfg.namespace}
        kubectl rollout status deploy/${cfg.deploy} -n ${cfg.namespace}
    """
}
```

```groovy
// Jenkinsfile
@Library('my-shared-lib') _

pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                deployToK8s(deploy: 'api', container: 'api',
                            image: env.IMAGE, namespace: 'prod')
            }
        }
    }
}
```

Configure the library in **Manage Jenkins → System** with a Git source + default version.

## Multibranch Pipelines

Jenkins discovers branches + PRs in a repo and runs the `Jenkinsfile` from each one. No per-branch job config — config lives with code.

- Auto-create jobs for new branches.
- PR builds, including PR title in build cause.
- Branch indexing on a schedule + webhook for instant trigger.

## Blue Ocean / Modern UI

The classic Jenkins UI is rough. Blue Ocean gives a per-pipeline modern view. Largely abandoned by the project — many shops use the newer "pipeline graph" view instead.

## Common Production Patterns

- **Build container in K8s with kaniko or buildx** — no privileged Docker-in-Docker if you don't need it.
- **Restrict shell access** — devs commit Jenkinsfile, can't SSH the controller.
- **JCasC (Configuration as Code)** — `jenkins.yaml` defines users, plugins, agents. Reproducible controller.
- **Backup `$JENKINS_HOME`** — config, jobs, credentials all live here. Or, better, treat it as cattle: JCasC + image-based.

## Interview Questions

**Q: Declarative vs scripted pipeline — when?**
A: Declarative for 95% of cases — readable, validated, restartable, integrates with Blue Ocean / pipeline graph. Drop into `script {}` blocks for Groovy logic you can't express declaratively. Pure scripted is legacy.

**Q: Why ephemeral agents?**
A: Reproducibility (every build starts clean), scale (provision on demand), security (compromise of one build doesn't carry to the next), no manual agent maintenance.

**Q: What's a Shared Library and why use it?**
A: A Git repo of Groovy helpers loaded by Jenkinsfiles via `@Library`. DRY common logic (deploy, notify, build) across many repos. Versioned with Git — pin pipelines to a tag for stability.

**Q: How do you handle secrets?**
A: Jenkins Credentials store with `withCredentials` block. For cloud creds, prefer IAM Instance Profile (EC2 agent) or IRSA (EKS Pod agent) — no Jenkins-managed cred needed.

**Q: Pipeline fails halfway. How do you debug without rerunning the whole thing?**
A: Restart from stage: in classic UI or via "Replay" / "Restart from Stage". Replay also lets you edit the Jenkinsfile inline to try a fix before committing.

**Q: How do you keep Jenkins itself reproducible?**
A: JCasC defines config in YAML; install plugins from a manifest; treat the controller image as immutable; back up only the job/credentials state. Or use cloud-hosted equivalents (CloudBees, Jenkins Operator on K8s).

## Common Pitfalls

- Master/controller doing actual build work — kill performance and security.
- Hardcoding versions in many Jenkinsfiles instead of using a Shared Library.
- Privileged DinD agents — easy lateral move for an attacker.
- Storing `JENKINS_HOME` on local disk with no backup — single-point-of-loss for jobs + creds.
- Ignoring plugin CVEs — Jenkins is high-value target; update monthly minimum.
- Free-style jobs that aren't in git — config drift, no review, vanish if the controller dies.

## Related

- [10-ci-cd-fundamentals.md](10-ci-cd-fundamentals.md)
- [11-github-actions.md](11-github-actions.md)
- [21-security-devsecops.md](21-security-devsecops.md)
