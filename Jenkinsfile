// GitOps pipeline: build -> push image -> render Helm chart -> commit to the GitOps repo.
// Jenkins never touches the cluster. Argo CD deploys whatever is in:
//   https://github.com/elevy99927-demos/gitops  ->  systems/<DEPARTMENT>/<ENV>/<user>.yaml

properties([
  parameters([
    choice(name: 'ENV', choices: ['dev', 'qa', 'prd'], description: 'Environment: systems/<DEPARTMENT>/<ENV>/'),
    choice(name: 'DEPARTMENT', choices: ['hr', 'finance', 'legal', 'digital', 'devops'], description: 'Department = folder in gitops = Kubernetes namespace'),
    booleanParam(name: 'SERVICE_MONITOR', defaultValue: false, description: 'Render the ServiceMonitor (needs Prometheus Operator CRDs)'),
    choice(name: 'DIND_STORAGE_DRIVER', choices: ['overlay2', 'vfs'], description: 'Use vfs only if overlay2 fails')
  ])
])

def user        = "per13"               // Docker Hub user (lowercase) = gitops file name = release name
def appname     = "leavetracker"           // App name from the class sheet (LeaveTracker), lowercase for Docker Hub
def appimage    = "${user}/${appname}"
def apptag      = "${env.BUILD_NUMBER}"
def envName     = params.ENV ?: 'dev'
def department  = params.DEPARTMENT ?: 'hr'
def serviceMon  = params.SERVICE_MONITOR ? 'true' : 'false'
def storageDrv  = params.DIND_STORAGE_DRIVER ?: 'overlay2'

def gitopsRepo  = "github.com/elevy99927-demos/gitops.git"
def gitopsFile  = "systems/${department}/${envName}/${user}.yaml"

podTemplate(containers: [
      containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent', ttyEnabled: true),
      containerTemplate(name: 'deployer', image: 'elevy99927/k8s-deployer:latest', command: 'cat', ttyEnabled: true),
      containerTemplate(name: 'docker', image: 'docker:27-dind', privileged: true,
                        args: "--storage-driver=${storageDrv}",
                        envVars: [envVar(key: 'DOCKER_TLS_CERTDIR', value: '')],
                        resourceRequestCpu: '500m', resourceRequestMemory: '512Mi',
                        resourceLimitMemory: '2Gi')
  ],
  volumes: [
      emptyDirVolume(mountPath: '/var/lib/docker', memory: false)
  ]) {
    // Jenkins credentials:
    //   docker-cred  : Username with password - <user> + Docker Hub access token
    //   github-token : Username with password - GitHub user + token with write access to the gitops repo
    node(POD_LABEL) {
        stage('validate') {
            if (user == 'CHANGE_ME') {
                error("Set 'def user' in the Jenkinsfile to your Docker Hub user")
            }
            echo "Target: ${gitopsFile} (image ${appimage}:${apptag})"
        }

        stage('checkout') {
            container('jnlp') {
                sh '/usr/bin/git config --global http.sslVerify false'
                checkout scm
            }
        }

        stage('build') {
            container('docker') {
                sh '''
                    for i in $(seq 1 30); do docker info >/dev/null 2>&1 && exit 0; sleep 2; done
                    echo "docker daemon did not start"; docker info; exit 1
                '''
                withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_TOKEN')]) {
                    sh 'echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin'
                }
                sh "docker build --network=host -t ${appimage}:${apptag} -t ${appimage}:latest \"${env.WORKSPACE}\""
            }
        }

        stage('push') {
            container('docker') {
                sh """
                    set -e
                    docker push ${appimage}:${apptag}
                    docker push ${appimage}:latest
                    docker logout
                """
            }
        }

        stage('gitops-commit') {
            container('deployer') {
                withCredentials([usernamePassword(credentialsId: 'github-token', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh """#!/bin/sh
                        set -e
                        command -v git >/dev/null || apk add --no-cache git

                        rm -rf gitops
                        git clone https://\${GIT_USER}:\${GIT_TOKEN}@${gitopsRepo} gitops
                        cd gitops
                        git config user.name  "Jenkins (${user})"
                        git config user.email "${user}@users.noreply.github.com"

                        if [ ! -d "systems/${department}/${envName}" ]; then
                            echo "systems/${department}/${envName} does not exist in the gitops repo"; exit 1
                        fi

                        helm template ${user} "${env.WORKSPACE}/chart" \\
                            --set image.repository=${appimage} \\
                            --set image.tag=${apptag} \\
                            --set serviceMonitor.enabled=${serviceMon} \\
                            > ${gitopsFile}

                        git add ${gitopsFile}
                        if git diff --cached --quiet; then
                            echo "No changes in ${gitopsFile} - nothing to deploy"
                            exit 0
                        fi
                        git commit -m "Deploy ${appimage}:${apptag} to ${department}/${envName} (build ${env.BUILD_NUMBER}) by ${user}"

                        for i in 1 2 3 4 5; do
                            if git push origin HEAD:main; then exit 0; fi
                            echo "push rejected, rebasing (attempt \$i)"
                            git pull --rebase origin main
                            sleep \$i
                        done
                        echo "push failed after 5 attempts"; exit 1
                    """
                }
            }
        }
    }
}
