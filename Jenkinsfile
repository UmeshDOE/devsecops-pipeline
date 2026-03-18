pipeline {
    agent any
    
    environment {
        // AWS Configuration - REPLACE WITH YOUR VALUES
        AWS_ACCOUNT_ID = '039108689342'
        AWS_REGION = 'us-east-1'
        ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/devsecops-app"
        
        // Docker Hub Configuration - REPLACE WITH YOUR VALUES
        DOCKER_HUB_USERNAME = 'your_dockerhub_username'
        DOCKER_HUB_REPO = "${DOCKER_HUB_USERNAME}/devsecops-app"
        
        // Application Configuration
        APP_NAME = 'devsecops-app'
        IMAGE_TAG = "${BUILD_NUMBER}-${GIT_COMMIT.substring(0,7)}"
        
        // SonarQube Configuration
        SONAR_HOST_URL = 'http://localhost:9000'
        SONAR_PROJECT_KEY = 'devsecops-app'
        SONAR_PROJECT_NAME = 'DevSecOps Application'
        
        // Scan Thresholds
        TRIVY_SEVERITY = 'HIGH,CRITICAL'
    }
    
    tools {
        jdk 'jdk17'
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/YOUR_USERNAME/devsecops-pipeline.git'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.projectName="${SONAR_PROJECT_NAME}" \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=**/*.js,**/node_modules/** \
                          -Dsonar.java.binaries=. \
                          -Dsonar.sourceEncoding=UTF-8
                    '''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '''
                    --scan ./app
                    --format HTML
                    --format JSON
                    --format XML
                    --out ./dependency-check-report
                    --failOnCVSS 7
                    --enableExperimental
                ''', odcInstallation: 'DP-Check'
                
                dependencyCheckPublisher pattern: 'dependency-check-report/dependency-check-report.xml'
                
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'dependency-check-report',
                    reportFiles: 'dependency-check-report.html',
                    reportName: 'OWASP Dependency Check Report'
                ])
            }
        }
        
        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    echo "Running Trivy filesystem scan..."
                    trivy fs --severity ${TRIVY_SEVERITY} --exit-code 0 --format table ./app || true
                    trivy fs --format json --output trivy-fs-report.json ./app || true
                '''
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    docker.build("${APP_NAME}:${IMAGE_TAG}", "./app")
                }
            }
        }
        
        stage('Docker Scout Scan') {
            steps {
                sh '''
                    echo "Running Docker Scout scan..."
                    docker scout quickview ${APP_NAME}:${IMAGE_TAG} || true
                    docker scout cves ${APP_NAME}:${IMAGE_TAG} --format sarif --output scout-report.sarif || true
                '''
            }
        }
        
        stage('Trivy Image Scan') {
            steps {
                sh '''
                    echo "Running Trivy image scan..."
                    trivy image --severity ${TRIVY_SEVERITY} --exit-code 0 --format table ${APP_NAME}:${IMAGE_TAG} || true
                    trivy image --format json --output trivy-image-report.json ${APP_NAME}:${IMAGE_TAG} || true
                '''
            }
        }
        
        stage('Push to Docker Hub') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry('', 'dockerhub-creds') {
                        def appImage = docker.image("${APP_NAME}:${IMAGE_TAG}")
                        appImage.push()
                        appImage.push('latest')
                        
                        // Also push with Docker Hub username
                        sh """
                            docker tag ${APP_NAME}:${IMAGE_TAG} ${DOCKER_HUB_REPO}:${IMAGE_TAG}
                            docker tag ${APP_NAME}:${IMAGE_TAG} ${DOCKER_HUB_REPO}:latest
                        """
                        docker.withRegistry('', 'dockerhub-creds') {
                            def hubImage = docker.image("${DOCKER_HUB_REPO}:${IMAGE_TAG}")
                            hubImage.push()
                            docker.image("${DOCKER_HUB_REPO}:latest").push()
                        }
                    }
                }
            }
        }
        
        stage('Push to AWS ECR') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        docker tag ${APP_NAME}:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}
                        docker tag ${APP_NAME}:${IMAGE_TAG} ${ECR_REPO}:latest
                        docker push ${ECR_REPO}:${IMAGE_TAG}
                        docker push ${ECR_REPO}:latest
                    """
                }
            }
        }
        
        stage('Deploy to Container') {
            steps {
                sh '''
                    # Stop and remove existing container if exists
                    docker stop ${APP_NAME} || true
                    docker rm ${APP_NAME} || true
                    
                    # Run new container
                    docker run -d \
                        --name ${APP_NAME} \
                        -p 80:80 \
                        --restart unless-stopped \
                        ${APP_NAME}:${IMAGE_TAG}
                    
                    echo "Container deployed successfully!"
                '''
            }
        }
        
        stage('Smoke Test') {
            steps {
                sh '''
                    # Wait for container to be ready
                    echo "Waiting for container to start..."
                    sleep 15
                    
                    # Test application with retries
                    for i in {1..5}; do
                        if curl -f http://localhost:80 > /dev/null 2>&1; then
                            echo "Smoke test passed! Application is responding."
                            exit 0
                        fi
                        echo "Attempt $i: Waiting for application..."
                        sleep 5
                    done
                    
                    echo "Smoke test failed! Application not responding."
                    exit 1
                '''
            }
        }
        
        stage('Upload Reports to S3') {
            steps {
                sh '''
                    # Create timestamp for report folder
                    TIMESTAMP=$(date +%Y%m%d-%H%M%S)
                    REPORT_FOLDER="reports/${BUILD_NUMBER}-${TIMESTAMP}"
                    
                    # Upload all reports to S3
                    aws s3 cp trivy-fs-report.json s3://${S3_BUCKET}/${REPORT_FOLDER}/trivy-fs-report.json || true
                    aws s3 cp trivy-image-report.json s3://${S3_BUCKET}/${REPORT_FOLDER}/trivy-image-report.json || true
                    aws s3 cp scout-report.sarif s3://${S3_BUCKET}/${REPORT_FOLDER}/scout-report.sarif || true
                    aws s3 cp dependency-check-report/ s3://${S3_BUCKET}/${REPORT_FOLDER}/dependency-check/ --recursive || true
                    
                    echo "Reports uploaded to s3://${S3_BUCKET}/${REPORT_FOLDER}/"
                '''
            }
        }
    }
    
    post {
        always {
            // Archive artifacts
            archiveArtifacts artifacts: '**/*.json,**/*.html,**/*.sarif,**/*.xml', allowEmptyArchive: true
            
            // Clean up Docker images to save space
            sh '''
                docker image prune -f
                docker container prune -f
            '''
            
            // Send email notification
            emailext (
                to: 'admin@example.com',
                subject: "Pipeline ${currentBuild.fullDisplayName} - ${currentBuild.result}",
                body: """
                    <h2>Pipeline Execution Summary</h2>
                    <p><b>Project:</b> DevSecOps Pipeline</p>
                    <p><b>Build Number:</b> ${BUILD_NUMBER}</p>
                    <p><b>Status:</b