def namespace = "default"
def chartName = "demo-rest-service"
def version = "1.0.6+5e750ab"
def chart = chartName + "-" + version + ".tgz"
def kustomizationPath = "k8s/${chartName}/templates/overlays/environments/dev"
def helmRepo = "https://axamit.jfrog.io/artifactory/helm-stable"

pipeline {

  environment {
    PROJECT = "gke-cluster-demo-1"
    CLUSTER = "prod-cluster"
    CLUSTER_ZONE = "northamerica-northeast1-a"
    JENKINS_CRED = "gke-cluster-demo"
    DOCKER_REGISTRY_SECRET = 'docker-hub-secret'
  }

  agent {
      kubernetes {
          yamlFile 'KubernetesPod.yaml'
      }
  }
  stages {
    stage('Download Artifact') {
      steps{
        echo "Downloading Helm chart..."
        withCredentials([usernamePassword(credentialsId: 'artifactory-secret',
                                        usernameVariable: 'HELM_STABLE_USERNAME',
                                        passwordVariable: 'HELM_STABLE_PASSWORD')]) {
          sh """
          wget --auth-no-challenge  --http-user=\${HELM_STABLE_USERNAME} --http-password=\${HELM_STABLE_PASSWORD} ${helmRepo}/${chart}
          tar -zxvf ${chart}
          """
        }
      }
    }
    stage('Prepare Deployment') {
      steps{
        echo "Rendering Helm templates..."

        container('helm') {
            sh """
             mkdir k8s
             cd ${chartName}
             helm template --debug . --output-dir ../k8s
             """
        }
      }
    }
    stage('Deploy Development') {
       steps {
           echo "Deploying..."

           container('kustomize') {
             sh """
             cd ${kustomizationPath}
             kustomize build . > deployment.yaml
             cat deployment.yaml
             """
           }
           /*
           container('kubectl') {

             step([$class: 'KubernetesEngineBuilder',
                    namespace: namespace,
                    projectId: env.PROJECT,
                    clusterName: env.CLUSTER,
                    zone: env.CLUSTER_ZONE,
                    manifestPattern: '${kustomizationPath}/deployment.yaml',
                    credentialsId: env.JENKINS_CRED,
                    verifyDeployments: false])
           }
           */
        }
    }

  }

}
