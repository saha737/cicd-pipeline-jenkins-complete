pipeline {
  agent any
  environment {
    DOCKER_IMAGE_NAME = 'sahasra/train-schedule'
  }

  stages {
        stage('Build') {
        agent {
            docker {
            image 'eclipse-temurin:8-jdk'   // required
            args  '-u 0:0'                  // optional
            }
        }
        steps {
            sh '''
            set -eux
            apt-get update
            apt-get install -y curl ca-certificates gnupg
            curl -fsSL https://deb.nodesource.com/setup_14.x | bash -
            apt-get install -y nodejs
            export ORG_GRADLE_PROJECT_nodeDownload=false
            export GRADLE_OPTS="$GRADLE_OPTS -Dcom.moowork.node.download=false"
            ./gradlew --no-daemon clean build
            '''
            archiveArtifacts artifacts: 'dist/trainSchedule.zip', allowEmptyArchive: true
        }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = 1
            }
            steps {
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube-canary.yml',
                    enableConfigSubstitution: true
                )
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = 0
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube-canary.yml',
                    enableConfigSubstitution: true
                )
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube.yml',
                    enableConfigSubstitution: true
                )
            }
        }
   }
}