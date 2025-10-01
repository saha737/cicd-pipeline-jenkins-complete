pipeline {
    agent any
    environment {
        //be sure to replace "bhavukm" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "sahasra/train-schedule"
    }
 stage('Build') {
   agent { docker { image 'eclipse-temurin:8-jdk' } } // JDK 8 for Gradle 4.6
    steps {
      sh '''
        set -eux
        # Install Node 14.x in the container
        apt-get update
        apt-get install -y curl ca-certificates gnupg
        curl -fsSL https://deb.nodesource.com/setup_14.x | bash -
        apt-get install -y nodejs
        node -v
        npm -v

       # Tell the (old) Gradle Node plugin to use system Node, not download it
       # These env -> Gradle project properties are safe to set even if one is ignored.
        export ORG_GRADLE_PROJECT_nodeDownload=false
        export GRADLE_OPTS="$GRADLE_OPTS -Dcom.moowork.node.download=false"

        ./gradlew --no-daemon clean build
      '''
      // If your artifact is a JAR/WAR, archive from build/libs:
      archiveArtifacts artifacts: 'build/libs/*', allowEmptyArchive: true
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
