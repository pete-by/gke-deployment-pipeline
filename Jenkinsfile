
/**
 Requirements:
 1. At each moment of time we want to know what application revision is deployed to each of the environments
 2. We want to track history of application revision promotion through environments
 3. We want to be able to to (re)deploy any application revision to an environment it is qualified (e.g. deploy to demo on demand)
 4. We want to be able to update deployment scripts without invoking redeployment
*/

/*

 1. Checkout from current branch (done by Jenkins)
 2. If it is not tagged skip all stages this is to avoid redeployment if Jenkinsfile or KubernetesPod.yaml are updated
 2. If previous stage was approved (manually or automatically) set next stage
 3. Fill release-info.yaml
 4. Make a deployment
 5. Commit changes to release-info.yaml to a proper branch linked to a next stage
 */

def writeReleaseInfo(info) {
  def releaseInfo = info
  writeYaml file: 'release-info.yaml', data: releaseInfo
}

def getBranchForStage(stage) {
  def branch = ($targetStage == 'prod') ? 'master' : $targetStage // master->prod, dev->dev, test->test
  branch
}

def STAGES = [
   dev : [project: "gke-cluster-demo-1", cluster: "dev-cluster", clusterZone: "northamerica-northeast1-a", credentialsId: "gke-cluster-demo"],
   test : [project: "gke-cluster-demo-1", cluster: "test-cluster", clusterZone: "northamerica-northeast1-a", credentialsId: "gke-cluster-demo"],
   staging : [project: "gke-cluster-demo-1", cluster: "staging-cluster", clusterZone: "northamerica-northeast1-a", credentialsId: "gke-cluster-demo"],
   prod : [project: "gke-cluster-demo-1", cluster: "prod-cluster", clusterZone: "northamerica-northeast1-a", credentialsId: "gke-cluster-demo"]
]

def RELEASE_INFO_FILENAME = "release-info.yaml"
def commitId
def commitRevision
def namespace = "default"

def appVersion
def appTitle= "Demo Rest Service"
def appName = "demo-rest-service"
def appGitRepo = "git@github.com:pete-by/demo-rest-service.git"

def chartName = appName
def helmRepo = "https://axamit.jfrog.io/artifactory/helm-stable"

def stageName
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
                    def releaseInfo = readYaml file: "$RELEASE_INFO_FILENAME"
                    stageName = env.BRANCH_NAME
                    targetStage = STAGES[stageName] // stage environment to deploy to

                    commitId = sh "git rev-parse HEAD"
                    try {
                        // get a release version (revision) if it is associated with the commit
                        commitRevision = sh returnStdout: true, script: "git describe --exact-match --tags $commitId 2> /dev/null || echo ''"
                        commitRevision = commitRevision.trim() // to trim new lines
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

                    def chart = chartName + "-" + version + ".tgz"
                    echo "Downloading Helm chart..."
                    withCredentials([usernamePassword(credentialsId: 'artifactory-secret',
                                                    usernameVariable: 'HELM_STABLE_USERNAME',
                                                    passwordVariable: 'HELM_STABLE_PASSWORD')]) {
                        sh """
                           wget --auth-no-challenge  --http-user=\${HELM_STABLE_USERNAME} --http-password=\${HELM_STABLE_PASSWORD} ${helmRepo}/${chart}
                           tar -zxvf ${chart}
                           """
                    }

                    container('helm') {
                        sh """
                           mkdir k8s
                           cd ${chartName}
                           helm template --debug . --output-dir ../k8s
                           """
                    }

                    echo "Rendering Kustomize config..."
                    def kustomizationPath = "k8s/${chartName}/templates/overlays/environments/$targetStage"
                    container('kustomize') {
                        sh """
                           cd ${kustomizationPath}
                           kustomize build . > deployment.yaml
                           cat deployment.yaml
                           """
                    }

                } // script

            } // steps
        } // stage

        stage('Deployment') {
            steps {

                script {
                    if(targetStage && commitRevision) { // deploy if stage exists, otherwise skip
                        // TODO use locks https://plugins.jenkins.io/lockable-resources
                        echo "Deploying to $stageName"
                        container('kubectl') {
                            step([$class: 'KubernetesEngineBuilder',
                                namespace: namespace,
                                projectId: s.project,
                                clusterName: s.cluster,
                                zone: s.clusterZone,
                                manifestPattern: '${kustomizationPath}/deployment.yaml',
                                credentialsId: s.credentialsId,
                                verifyDeployments: false])
                        }

                    } else {
                        if(!targetStage) {
                            echo 'Skipping deployment as there is no stage environment to deploy to'
                        }
                        if(!commitRevision) {
                            echo 'Commit is not tagged with a release revision, deployment will be skipped'
                        }
                    }
                }
            } // steps

        } // stage

        stage('Validation') {

            echo "Deployment validation..."

            script {

                def nextStage
                switch (stageName) {
                    case 'dev' : nextStage = 'prod'
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

                timeout(time: 30, unit: 'MINUTES') {
                    input(id: "Deploy Gate", message: "Deploy to $nextStage?", ok: 'Deploy')
                    // commit release info to next stage branch to trigger deployment
                    def branch = getBranchForStage(nextStage)
                    sh """
                    git checkout $branch
                    git cherry-pick $commitId
                    git tag -a $commitRevision -m 'Jenkins Deployment Agent'
                    git push --atomic --tags -u origin $branch
                    """
                }
            }

        }

    } // stages

} // pipeline
