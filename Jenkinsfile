pipeline {
    agent any
    
    environment {
        // AWS Configuration
        AWS_ACCOUNT_ID = '039108689342'
        AWS_REGION = 'us-east-1'
        ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/devsecops-app"
        
        // Docker Hub Configuration
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
    
    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/UmeshDOE/devsecops-pipeline.git'
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
        
        stage('Trivy Image Scan') {
            steps {
                sh '''
                    echo "Running Trivy image scan..."
                    trivy image --severity ${TRIVY_SEVERITY} --exit-code 0 --format table ${APP_NAME}:${IMAGE_TAG} || true
                    trivy image --format json --output trivy-image-report.json ${APP_NAME}:${IMAGE_TAG} || true
                '''
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
                    docker stop ${APP_NAME} || true
                    docker rm ${APP_NAME} || true
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
                    echo "Waiting for container to start..."
                    sleep 15
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
                    TIMESTAMP=$(date +%Y%m%d-%H%M%S)
                    REPORT_FOLDER="reports/${BUILD_NUMBER}-${TIMESTAMP}"
                    aws s3 cp trivy-fs-report.json s3://${S3_BUCKET}/${REPORT_FOLDER}/trivy-fs-report.json || true
                    aws s3 cp trivy-image-report.json s3://${S3_BUCKET}/${REPORT_FOLDER}/trivy-image-report.json || true
                    aws s3 cp dependency-check-report/ s3://${S3_BUCKET}/${REPORT_FOLDER}/dependency-check/ --recursive || true
                    echo "Reports uploaded to s3://${S3_BUCKET}/${REPORT_FOLDER}/"
                '''
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: '**/*.json,**/*.html,**/*.sarif,**/*.xml', allowEmptyArchive: true
            sh 'docker image prune -f || true'
            sh 'docker container prune -f || true'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed! Check logs for details.'
        }
    }
}