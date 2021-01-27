/*
 Requirements:
 1. We want to be able to deploy to different environments and be able to add new or remove old environments
 2. At each moment of time we want to know what application revision is deployed to each of the environments
 3. We want to track history of application revision promotion through environments
 4. We want to be able to to (re)deploy any application revision to an environment it is qualified (e.g. deploy to demo on demand)
 5. We want to be able to update deployment scripts without invoking redeployment
*/

/*
 1. Checkout from current branch (done by Jenkins)
 2. If it is not tagged skip all stages this is to avoid redeployment if Jenkinsfile or KubernetesPod.yaml are updated
 2. If previous stage was approved (manually or automatically) set next stage
 3. Make a deployment
 4. Merge release-info.yaml to a proper branch linked to a next stage
 */

def STAGES = [
   dev : [project: "gke-cluster-demo-1", cluster: "dev-cluster", clusterZone: "northamerica-northeast1-a", credentialsId: "gke-cluster-demo", skipDeploy: true],
   test : [project: "gke-cluster-demo-1", cluster: "test-cluster", clusterZone: "northamerica-northeast1-a", credentialsId: "gke-cluster-demo", skipDeploy: true],
   staging : [project: "gke-cluster-demo-1", cluster: "staging-cluster", clusterZone: "northamerica-northeast1-a", credentialsId: "gke-cluster-demo", skipDeploy: true],
   prod : [project: "gke-cluster-demo-1", cluster: "prod-cluster", clusterZone: "northamerica-northeast1-a", credentialsId: "gke-cluster-demo"]
]

def GITHUB_SSH_SECRET = 'github-ssh-secret'
def RELEASE_BRANCH_NAME = 'master'
def RELEASE_INFO_FILENAME = "release-info.yaml"
def commitId
def releaseTag
def namespace = "default"

def releaseInfo
def appVersion
def appTitle= "Demo Rest Service"
def appName = "demo-rest-service"
def appGitRepo = "git@github.com:pete-by/demo-rest-service.git"

def helmRepo = "https://axamit.jfrog.io/artifactory/helm-stable"

def stageName // name of a stage environment, e.g. dev or prod
def targetStage

pipeline {

    environment {
        DOCKER_REGISTRY_SECRET = 'docker-hub-secret'
    }

    agent {
        kubernetes {
            yamlFile 'KubernetesPod.yaml'
        }
    }
    stages {
        stage('Initialization') {
            steps {
                echo "Initialization..."
                script {
                    releaseInfo = readYaml file: "$RELEASE_INFO_FILENAME"
                    appVersion = releaseInfo.version
                    stageName = getStageForBranch(env.BRANCH_NAME) // get stage for current branch
                    targetStage = STAGES[stageName] // stage environment to deploy to

                    commitId = sh returnStdout: true, script: 'git rev-parse HEAD'
                    try {
                        // get a release number (revision) if it is associated with the commit
                        releaseTag = sh returnStdout: true, script: "git describe --exact-match --tags $commitId 2> /dev/null || echo ''"
                        releaseTag = releaseTag.trim() // to trim new lines
                    } catch(err) {
                        println("Change detected is not tagged with a release revision, deployment will be skipped")
                    }

                }
            } // steps
        } // stage

        stage('Prepare Deployment') {
            steps {

                echo "Rendering Helm templates..."

                script {

                    if(targetStage) {

                        def module = releaseInfo.modules[0]
                        def artifact = module.artifacts[0]

                        echo "Downloading Helm chart..."

                        withCredentials([usernamePassword(credentialsId: 'artifactory-secret',
                                                        usernameVariable: 'HELM_STABLE_USERNAME',
                                                        passwordVariable: 'HELM_STABLE_PASSWORD')]) {
                            sh """
                               wget --auth-no-challenge  --http-user=\${HELM_STABLE_USERNAME} --http-password=\${HELM_STABLE_PASSWORD} ${artifact.url}
                               mkdir -p ${artifact.name} && tar -zxvf ${artifact.filename}
                               """
                        }

                        container('helm') {
                            sh """
                               mkdir k8s
                               cd ${artifact.name}
                               helm template --debug . --output-dir ../k8s
                               """
                        }

                        echo "Rendering Kustomize config..."
                        def kustomizationPath = "k8s/${artifact.name}/templates/overlays/environments/$stageName"
                        container('kustomize') {
                            sh """
                               cd ${kustomizationPath}
                               kustomize build . > deployment.yaml
                               """
                        }

                        // Cleanup
                        echo "Deleting artifact files"
                        sh """
                        rm -rf ${env.WORKSPACE}/${artifact.filename}
                        rm -rf ${env.WORKSPACE}/${artifact.name}
                        """

                    }

                } // script

            } // steps
        } // stage

        stage('Deployment') {
            steps {

                script {
                    // deploy if stage exists, otherwise skip
                    def canDeploy = targetStage && !targetStage.skipDeploy && (env.BRANCH_NAME != RELEASE_BRANCH_NAME || releaseTag) // to prevent non-tagged release branch deployment

                    if(canDeploy) {
                        // TODO use locks https://plugins.jenkins.io/lockable-resources
                        echo "Deploying to $stageName"
                        container('kubectl') {
                            step([$class: 'KubernetesEngineBuilder',
                                namespace: namespace,
                                projectId: targetStage.project,
                                clusterName: targetStage.cluster,
                                zone: targetStage.clusterZone,
                                manifestPattern: '${kustomizationPath}/deployment.yaml',
                                credentialsId: targetStage.credentialsId,
                                verifyDeployments: false])
                        }

                        sh "rm -rf ${env.WORKSPACE}/k8s"

                    } else {
                       echo 'Skipping deployment as there is no stage environment to deploy to'
                    }
                }
            } // steps

        } // stage

        stage('Validation') {

            steps {
                echo "Deployment validation..."

                script {

                    def nextStage
                    switch (stageName) {
                        case 'dev' : nextStage = 'test'
                                     break // just prod because of short pipeline
                        case 'test' : nextStage = 'staging'
                                      break
                        case 'staging' : nextStage = 'prod'
                                         break
                        case 'prod' : nextStage = ''
                                      break // we do not deploy anywhere else after prod
                        default : nextStage = 'dev'
                                  break
                    }

                    if(nextStage) {
                        timeout(time: 30, unit: 'MINUTES') {
                            input(id: "Deploy Gate", message: "Deploy to $nextStage?", ok: 'Deploy')
                            // commit release-info.yaml to next stage branch to trigger deployment
                            def branch = getBranchForStage(nextStage)
                            // --allow-empty is to allow re-running the same build without failing
                            sh """
                                git checkout $branch
                                git checkout --theirs origin/${env.BRANCH_NAME} -- release-info.yaml
                                git commit --allow-empty -m 'Jenkins Deployment Agent [${env.BRANCH_NAME} build #${BUILD_NUMBER}]: merged release-info.yaml'
                            """

                            releaseTag = releaseInfo.vcs.release                            
                            if(branch == RELEASE_BRANCH_NAME && releaseTag) {
                                echo "Tagging release $releaseTag"
                                sh "git tag -f -a $releaseTag -m 'Jenkins Deployment Agent [${env.BRANCH_NAME} build #${BUILD_NUMBER}]: $releaseTag released'"
                            }

                            sshagent(credentials: [GITHUB_SSH_SECRET]) {
                                sh "git push --atomic --tags -u origin $branch"
                            }
                        }
                    }
                }
            }
        }

    } // stages

} // pipeline


def writeReleaseInfo(info) {
  def releaseInfo = info
  writeYaml file: 'release-info.yaml', data: releaseInfo
}

def getBranchForStage(stageName) {
  def branchName = (stageName == 'prod') ? 'master' : stageName // master->prod, dev->dev, test->test
  branchName
}

def getStageForBranch(branchName) {
  def stageName = (branchName == 'master') ? 'prod' : branchName // prod->master, dev->dev, test->test
  stageName
}
