def namespace = "default"
def chartName = "demo-rest-service"
def version = "1.0.5+c00f7f5"
def chart = chartName + "-" + version + ".tgz"

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
             cd ${chartName}
             helm template --debug . --output-dir k8s
             ls
             """
        }
      }
    }
    stage('Deploy Development') {
       steps {
       }

        /*
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
