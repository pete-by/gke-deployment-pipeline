pipeline {

  environment {
    PROJECT = "gke-cluster-demo"
    CLUSTER = "jenkins-cd"
    CLUSTER_ZONE = "us-east1-d"
    JENKINS_CRED = "gke-cluster-demo"
  }

  def namespace = "default"
  def chartName = "demo-rest-service"
  def version = ""
  def chart = chartName + version + ".tgz"

  environment {
    DOCKER_REGISTRY_SECRET = 'docker-hub-secret'
  }
  agent {
      kubernetes {
          yamlFile 'KubernetesPod.yaml'
      }
  }
  stages {

    stage('Deploy Production') {

      steps{

        sh """
        wget https://axamit.jfrog.io/artifactory/helm-stable/${chart}
        tar -zxvf ${chart}
        """

        container('helm') {

        }

        container('kubectl') {
          step([$class: 'KubernetesEngineBuilder',
                namespace: namespace,
                projectId: env.PROJECT,
                clusterName: env.CLUSTER,
                zone: env.CLUSTER_ZONE,
                manifestPattern: 'k8s/services',
                credentialsId: env.JENKINS_CRED,
                verifyDeployments: false])
        }
      }

    }

  }

}
