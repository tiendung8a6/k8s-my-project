pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'nodejs25'
    }

    environment {
        SCANNER_HOME     = tool 'sonar-scanner'
        DOCKER_IMAGE     = 'latishisbumanglagcfdsa/my-app:latest'
        EKS_CLUSTER_NAME = 'my-cluster-eks'
        AWS_REGION       = 'ap-southeast-1'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/tiendung8a6/k8s-my-project.git'
                sh 'ls -la'   // debug nhanh
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarQube-servers') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=my-project \
                            -Dsonar.projectName=my-project
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true, credentialsId: 'Sonarqube-Token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('my-app') {
                    sh '''
                        ls -la
                        if [ ! -f package.json ]; then
                            echo "❌ package.json not found in my-app/"
                            exit 1
                        fi

                        rm -rf node_modules package-lock.json || true
                        npm ci --prefer-offline --no-audit --progress=false
                    '''
                }
            }
        }

        stage('Security Scans - OWASP Dependency-Check') {
            steps {
                dependencyCheck(
                    additionalArguments: '--scan ./ --format XML --disableYarnAudit --disableNodeAudit --noupdate',
                    odcInstallation: 'dependency-check'
                )
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Security Scan - Trivy FS') {
            steps {
                sh 'trivy fs --exit-code 0 --no-progress . > trivy-fs-report.txt'
                archiveArtifacts artifacts: 'trivy-fs-report.txt', allowEmptyArchive: true
            }
        }

        stage('Build & Push Docker Image') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh '''
                            docker build --no-cache -t ${DOCKER_IMAGE} -f my-app/Dockerfile ./my-app
                            docker push ${DOCKER_IMAGE}
                        '''
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                withAWS(region: "${AWS_REGION}", credentials: 'aws-credentials-id') {  // thay bằng credential thật của bạn
                    sh '''
                        echo "→ Updating kubeconfig for EKS cluster..."
                        aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}

                        echo "→ Current kubectl context:"
                        kubectl config current-context

                        echo "→ Applying Kubernetes manifests..."
                        kubectl apply -f deployment.yml --namespace default || true
                        kubectl apply -f service.yml     --namespace default || true

                        echo "→ Waiting for rollout..."
                        kubectl rollout status deployment/my-app --timeout=120s || true

                        echo "→ Current status:"
                        kubectl get pods,svc -l app=my-app
                    '''
                }
            }
        }
    }

    post {
        always {
            emailext (
                subject: "${currentBuild.currentResult} - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <p>Project: ${env.JOB_NAME}</p>
                    <p>Build: ${env.BUILD_NUMBER}</p>
                    <p>Status: ${currentBuild.currentResult}</p>
                    <p>Duration: ${currentBuild.durationString}</p>
                    <p>URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                """,
                to: 'tiendung8a6@gmail.com',
                attachLog: true,
                attachmentsPattern: 'trivy-fs-report.txt, **/dependency-check-report.xml',
                compressLog: true
            )

            // Có thể thêm cleanWs() ở đây nếu muốn dọn workspace sau mỗi build
        }

        success {
            echo "Pipeline completed successfully ✓"
        }

        failure {
            echo "Pipeline failed ✗"
        }
    }
}