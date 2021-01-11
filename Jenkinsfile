def namespace = "default"
def chartName = "demo-rest-service"
def version = "1.0.5+c00f7f5"
def chart = chartName + version + ".tgz"

pipeline {

  environment {
    PROJECT = "gke-cluster-demo-1"
    CLUSTER = "demo-prod"
    CLUSTER_ZONE = "northamerica-northeast1"
    JENKINS_CRED = "gke-cluster-demo"
    DOCKER_REGISTRY_SECRET = 'docker-hub-secret'
  }

  agent {
      kubernetes {
          yamlFile 'KubernetesPod.yaml'
      }
  }
  stages {

    stage('Deploy Development') {

      steps{

        sh """
        wget https://axamit.jfrog.io/artifactory/helm-stable/${chart}
        tar -zxvf ${chart}
        ls
        """

        /*
        container('helm') {
            "sh helm template --debug . --output-dir k8s"
        }

        container('kubectl') {
          step([$class: 'KubernetesEngineBuilder',
                namespace: namespace,
                projectId: env.PROJECT,
                clusterName: env.CLUSTER,
                zone: env.CLUSTER_ZONE,
                manifestPattern: 'k8s/demo-rest-service/templates/overlays/environments/dev',
                credentialsId: env.JENKINS_CRED,
                verifyDeployments: false])
        }
        */
      }

    }

  }

}
