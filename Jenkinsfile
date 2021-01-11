def namespace = "default"
def chartName = "demo-rest-service"
def version = "1.0.6+5e750ab"
def chart = chartName + "-" + version + ".tgz"

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

        withCredentials([usernamePassword(credentialsId: 'artifactory-secret', usernameVariable: 'HELM_STABLE_USERNAME', passwordVariable: 'HELM_STABLE_PASSWORD')]) {
            sh """
            wget --auth-no-challenge  --http-user=\${HELM_STABLE_USERNAME} --http-password=\${HELM_STABLE_PASSWORD} https://axamit.jfrog.io/artifactory/helm-stable/${chart}
            tar -zxvf ${chart}
            ls
            """
        }
      }
    }
    stage('Prepare Deployment') {
      steps{
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

           container('kubectl') {
             sh """
             ls -LR k8s
             cd ./k8s/demo-rest-service/templates/overlays/envronments/dev
             kubectl kustomize build . > deployment.yaml
             """

             step([$class: 'KubernetesEngineBuilder',
                    namespace: namespace,
                    projectId: env.PROJECT,
                    clusterName: env.CLUSTER,
                    zone: env.CLUSTER_ZONE,
                    manifestPattern: 'k8s/demo-rest-service/templates/overlays/envronments/dev/deployment.yaml',
                    credentialsId: env.JENKINS_CRED,
                    verifyDeployments: false])
            }

        }
    }

  }

}
