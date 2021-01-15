
/**
 Requirements:
 1. At each moment of time we want to know what application revision is deployed to each of the environments
 2. We want to track history of application revision promotion through environments
 3. We want to be able to to (re)deploy any application revision to an environment it is qualified (e.g. deploy to demo on demand)
*/

/*
 1. check if we have build info annotated with the project revision
 2. no -> 3. yes -> 5
 3. clone project and get tag for a project revision
 4. generate artefact name -> 7, set initial stage
 5. checkout
 6. set next stage
 7. fill build-info.yaml
 8. make a deployment
 9. commit changes to build-info.yaml
 */

def getLatestRevisionTagFromGit() {
    def defaultRevisionTag = '1.0.0'
    def latestRevisionTag = sh returnStdout: true, script: "git describe --match=*.*.* --abbrev=0 2> /dev/null || echo ${defaultRevisionTag}"
    latestRevisionTag?.trim()
}

def writeReleaseInfo(info) {
  echo info.inspect()
}

def STAGES = [
   dev : [project: "gke-cluster-demo-1", cluster: "dev-cluster", clusterZone: "northamerica-northeast1-a", credentialsId: "gke-cluster-demo"],
   test : [project: "gke-cluster-demo-1", cluster: "test-cluster", clusterZone: "northamerica-northeast1-a", credentialsId: "gke-cluster-demo"],
   staging : [project: "gke-cluster-demo-1", cluster: "staging-cluster", clusterZone: "northamerica-northeast1-a", credentialsId: "gke-cluster-demo"],
   prod : [project: "gke-cluster-demo-1", cluster: "prod-cluster", clusterZone: "northamerica-northeast1-a", credentialsId: "gke-cluster-demo"]
]

def appGitRevision = REVISION // Git revision 7 digit or full of an application
def appGitRevisionShort = appGitRevision.substring(0, 7) // Git revision 7 digit

def RELEASE_INFO_FILENAME = "release-info.yaml"
def commitId // If the file exists, will contain release-info.yaml commit id

def commitRevision
def namespace = "default"

def appVersion
def appTitle= "Demo Rest Service"
def appName = "demo-rest-service"
def appGitRepo = "git@github.com:pete-by/demo-rest-service.git"

def chartName = appName
def helmRepo = "https://axamit.jfrog.io/artifactory/helm-stable"

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
                    // Check if we have a release-info commit tagged with the application git revision hash,
                    // we assume that release-info.yaml is there, but we can check it
                    try {
                        commitId = sh returnStdout: true, script: "git rev-list -n 1 $appGitRevisionShort 2> /dev/null || echo ''"
                        commitId = commitId.trim()
                    } catch(err) {
                        echo "Cannot find release-info.yaml: $err"
                    }


                    if(!commitId) { // we haven't deployed corresponding application version yet

                        targetStage = "dev"; // dev or feature (on-demand)

                        // Checkout application revision to obtain tag and full version of an artifact /
                        sh "mkdir $appName" // create a target folder for checkout
                        dir(appName) {
                            checkout([$class: 'GitSCM', branches: [[name: appGitRevision]],
                                userRemoteConfigs: [[credentialsId: 'github-secret', url: appGitRepo]]])
                            appVersion = getLatestRevisionTagFromGit() + "-" + appGitRevisionShort
                            echo "version: $appVersion"
                        }
                        sh "rm -rf ./$appName" // clean up

                        // Create new Release Info
                        sh "git checkout -b $appVersion" // create a new branch for the application revision

                        writeReleaseInfo([
                            version: appVersion, stage: targetStage,
                            vcs: [revision: appGitRevision, url: appGitRepo],
                            modules: [[name: appName,
                                       artifacts: [name: appName + "-chart", type: "helm", sha1: "TODO", md5: "TODO"]]]
                        ]);

                    } else { // the corresponding application version was deployed before at least once

                        def releaseInfo = readYaml file: "$RELEASE_INFO_FILENAME"
                        if(releaseInfo.status == "approved") {
                            // Determine next stage
                            def currentStage
                            for (s in stages) {
                                if(currentStage) {
                                    targetStage = s // update next stage
                                    break
                                } else if(s.key == targetStage) {
                                    currentStage = targetStage
                                }
                            }
                        } else {
                            targetStage = "";
                        }

                        writeReleaseInfo(releaseInfo);
                    } // if
                }

            } // steps
        } // stage

        stage('Prepare Deployment') {
            steps {

                echo "Rendering Helm templates..."
               /*
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
                */
            } // steps
        } // stage
        stage('Deploying') {
            steps {

                echo "Deploying..."
                //echo "Deploying to $targetStage..."
                /*
                // TODO use locks https://plugins.jenkins.io/lockable-resources
                script {


                    container('kubectl') {
                        def s = STAGES[targetStage]
                        step([$class: 'KubernetesEngineBuilder',
                            namespace: namespace,
                            projectId: s.project,
                            clusterName: s.cluster,
                            zone: s.clusterZone,
                            manifestPattern: '${kustomizationPath}/deployment.yaml',
                            credentialsId: s.credentialsId,
                            verifyDeployments: false])
                    }

                    // Write status to release-info.yaml and target stage branch
                    echo "Saving deployment information..."
                    sh "git commit -m \"Deployed release $appVersion\""
                    sh "git push"
                    commitId = sh "git rev-parse HEAD"
                    sh "git checkout -b $targetStage"
                    sh "git cherry-pick $commitId"
                    sh "git push"
                }
                */
            } // steps

        } // stage
    } // stages

} // pipeline
