pipeline {
agent any

environment {
    DOCKER_USER    = "srinivasu56"
    IMAGE_NAME     = "saloon"
    IMAGE_TAG      = "latest"
    CONTAINER_NAME = "saloon-container"
    DOCKER_CREDS   = "docker-cred"
}

stages {

    stage('Checkout Code') {
        steps {
            checkout scmGit(
                branches: [[name: '*/main']],
                userRemoteConfigs: [[
                    credentialsId: 'git-cred',
                    url: 'https://github.com/Srinivasu2000/Saloon.git'
                ]]
            )
        }
    }

    stage('Build Application') {
        steps {
            sh '''
                echo "Building Maven project..."
                mvn clean package -DskipTests
            '''
        }
    }

  stage('Upload Artifact to Nexus') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'nexus-cred',
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS'
        )]) {

            sh '''
                echo "Uploading Artifact to Nexus..."

                ARTIFACT=$(ls target/*.jar | head -1)

                curl -v \
                -u "$NEXUS_USER:$NEXUS_PASS" \
                --upload-file "$ARTIFACT" \
                http://localhost:8081/repository/saloon/$(basename "$ARTIFACT")
            '''
        }
    }
}


stage('SonarQube Analysis') {
    steps {
        script {
            def scannerHome = tool 'sonar-scanner'

            withCredentials([string(
                credentialsId: 'sonar-token',
                variable: 'SONAR_TOKEN'
            )]) {

                withSonarQubeEnv('sonar-server') {

                    sh """
                        docker rm -f sonarcont || true

                        docker run -d \
                        --name sonarcont \
                        -p 9000:9000 \
                        sonarqube:latest

                        echo "Waiting for SonarQube..."

                        timeout=300
                        elapsed=0

                        until curl -s http://localhost:9000 >/dev/null; do
                            sleep 10
                            elapsed=\$((elapsed+10))

                            if [ \$elapsed -ge \$timeout ]; then
                                echo "SonarQube failed to start"
                                exit 1
                            fi
                        done

                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=saloon \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://localhost:9000 \
                        -Dsonar.token=\$SONAR_TOKEN
                    """
                }
            }
        }
    }
}

stage('Quality Gate') {
    steps {
        timeout(time: 10, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}





    

 
    stage('Build Docker Image') {
        steps {
            sh '''
                echo "Building Docker Image..."
                docker build -t ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG} .
            '''
        }
    }

    stage('Push Docker Image') {
        steps {
            withCredentials([
                usernamePassword(
                    credentialsId: 'docker-cred',
                    usernameVariable: 'DOCKER_HUB_USER',
                    passwordVariable: 'DOCKER_HUB_PASS'
                )
            ]) {

                sh '''
                    echo "$DOCKER_HUB_PASS" | docker login -u "$DOCKER_HUB_USER" --password-stdin
                    docker push ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }
    }

   stage('Deploy to Kubernetes') {
    steps {
        sh '''
            echo "Deploying to EKS..."

            if aws eks describe-cluster --region ap-southeast-2 --name salooncluster >/dev/null 2>&1; then
                aws eks update-kubeconfig --region ap-southeast-2 --name salooncluster

                kubectl get nodes
                kubectl apply -f Kubernetes/deploymentfile.yml
                kubectl apply -f Kubernetes/service.yml
            else
                echo "Cluster salooncluster does not exist. Skipping deployment."
            fi
        '''
    }
}

}
post {
    success {
        echo 'Pipeline executed successfully!'
    }

    failure {
        echo 'Pipeline failed. Check logs for details.'
    }
}

}
