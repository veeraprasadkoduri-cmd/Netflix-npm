pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node23'
    }

    environment {
        // Docker
        DOCKER_HUB   = 'veeradockerhub'
        IMAGE_NAME   = 'netflix-clone'
        IMAGE_TAG    = "${BUILD_NUMBER}"
        DOCKER_IMAGE = "${DOCKER_HUB}/${IMAGE_NAME}"

        // AWS / EKS
        AWS_REGION   = 'us-east-1'
        CLUSTER_NAME = 'mycluster'

        // Kubernetes Files
        KUBE_DEPLOY_PATH  = 'Kubernetes/deployment.yml'
        KUBE_SERVICE_PATH = 'Kubernetes/service.yml'

        // Mail
        RECIPIENTS = 'veeraprasad.koduri@gmail.com'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                url: 'https://github.com/veeraprasadkoduri-cmd/Netflix-npm.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build Application') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Test') {
            steps {
                sh '''
                if npm run | grep -q "test"; then
                    npm test -- --watchAll=false --passWithNoTests
                else
                    echo "No test script found. Skipping tests."
                fi
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'

                    withSonarQubeEnv('sq') {
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=netflix-clone \
                        -Dsonar.sources=src \
                        -Dsonar.projectName=Netflix-Clone \
                        -Dsonar.projectVersion=${BUILD_NUMBER}
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package Artifact') {
            steps {
                sh '''
                if [ -d dist ]; then
                    zip -r netflix-build.zip dist/
                else
                    zip -r netflix-build.zip build/
                fi
                '''
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-cred',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {

                    sh '''
                    curl -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file netflix-build.zip \
                    http://localhost:8081/repository/raw-hosted/netflix-build-${BUILD_NUMBER}.zip
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build -t $DOCKER_IMAGE:${BUILD_NUMBER} .
                docker tag $DOCKER_IMAGE:${BUILD_NUMBER} $DOCKER_IMAGE:latest
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    docker push $DOCKER_IMAGE:${BUILD_NUMBER}
                    docker push $DOCKER_IMAGE:latest
                    docker logout
                    '''
                }
            }
        }

        stage('Install Helm') {
            steps {
                sh '''
                if ! command -v helm >/dev/null 2>&1; then
                    curl -LO https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
                    tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
                    sudo mv linux-amd64/helm /usr/local/bin/helm
                    chmod +x /usr/local/bin/helm
                fi
                helm version
                '''
            }
        }

        stage('Setup Kubeconfig') {
            steps {
                sh '''
                aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
                kubectl get nodes
                '''
            }
        }

        stage('Deploy Monitoring') {
            steps {
                sh '''
                helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
                helm repo update

                helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
                --namespace monitoring \
                --create-namespace \
                --set grafana.service.type=LoadBalancer
                '''
            }
        }

        stage('Get Grafana Password') {
            steps {
                sh '''
                kubectl get secret monitoring-grafana \
                -n monitoring \
                -o jsonpath="{.data.admin-password}" | base64 --decode
                echo ""
                '''
            }
        }

        stage('Deploy Application to EKS') {
            steps {
                sh '''
                kubectl apply -f ${KUBE_DEPLOY_PATH}
                kubectl apply -f ${KUBE_SERVICE_PATH}
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                kubectl rollout status deployment/netflix-app --timeout=180s
                kubectl get pods -o wide
                kubectl get svc -o wide
                '''
            }
        }

        stage('Get Application URL') {
            steps {
                script {
                    def url = sh(
                        script: '''
                        kubectl get svc netflix-service \
                        -o jsonpath="{.status.loadBalancer.ingress[0].hostname}"
                        ''',
                        returnStdout: true
                    ).trim()

                    env.APP_URL = url
                    echo "Application URL: ${env.APP_URL}"
                }
            }
        }
    }

    post {

        success {
            emailext(
                to: "${RECIPIENTS}",
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
Pipeline SUCCESS 🎉

Application URL:
http://${env.APP_URL}

Build URL:
${env.BUILD_URL}
"""
            )
        }

        failure {
            emailext(
                to: "${RECIPIENTS}",
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
Pipeline FAILED ❌

Check Logs:
${env.BUILD_URL}
"""
            )
        }

        always {
            archiveArtifacts artifacts: 'netflix-build.zip', fingerprint: true
        }
    }
}
