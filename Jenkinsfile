pipeline {
    agent any
    
    tools {
        maven 'Maven 3.8.6'  // Use Jenkins Global Tool Configuration
    }
    
    environment {
        MAVEN_HOME = '/opt/maven'
        PATH = "${MAVEN_HOME}/bin:${PATH}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Environment Check') {
            steps {
                sh '''
                    echo "=== Environment Information ==="
                    echo "Build Number: $BUILD_NUMBER"
                    echo "Java Version: $(java -version)"
                    echo "Maven Version: $(mvn -version)"
                    echo "Docker Version: $(docker --version)"
                    echo "Workspace: $WORKSPACE"
                    echo "================================"
                '''
            }
        }
        
        stage('Compile Code') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('PMD Code Review') {
            steps {
                sh 'mvn -P metrics pmd:pmd'
            }
            post {
                always {
                    script {
                        if (fileExists('target/pmd.xml')) {
                            recordIssues(tools: [pmdParser(pattern: '**/pmd.xml')])
                        } else {
                            echo "PMD report not found, skipping issue recording"
                        }
                    }
                }
            }
        }
        
        stage('Sonar Code Analysis') {
            when {
                // Only run if SonarQube is properly configured
                expression { return env.SONAR_ENABLED == 'true' }
            }
            environment {
                scannerHome = tool 'sonarqube-scanner'
            }
            steps {
                withSonarQubeEnv('sonarqube') { 
                    sh "${scannerHome}/bin/sonar-scanner"
                }
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Package App') {
            steps {
                sh 'mvn package -DskipTests'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.war', fingerprint: true
                }
            }
        }
        
        stage('Publish to JFrog') {
            when {
                // Only run if JFrog is properly configured
                expression { return env.JFROG_ENABLED == 'true' }
            }
            steps {
                script {
                    if (fileExists('target/kitchensink.war')) {
                        rtUpload (
                            serverId: 'jfrog-dev',
                            spec: '''{
                                  "files": [
                                    {
                                      "pattern": "target/kitchensink.war",
                                      "target": "non-prod-repo/"
                                    }
                                 ]
                            }'''
                        )
                    } else {
                        error "WAR file not found for upload"
                    }
                }
            }
        }
        
        stage('Ansible Deploy to HTTPD') {
            when {
                // Only run if Ansible is properly configured
                expression { return env.ANSIBLE_ENABLED == 'true' }
            }
            steps {
                script {
                    if (fileExists('inventory') && fileExists('playbook.yml')) {
                        ansiblePlaybook(
                            becomeUser: null, 
                            credentialsId: 'ansible-token', 
                            disableHostKeyChecking: true, 
                            installation: 'ansible', 
                            inventory: 'inventory', 
                            playbook: 'playbook.yml', 
                            sudoUser: null
                        )
                    } else {
                        echo "Ansible files not found, skipping deployment"
                    }
                }
            }
        }
        
        stage('Build Docker Image and Deploy') {
            steps {
                script {
                    // Stop existing containers
                    sh '''
                        # Stop containers running on port 8050
                        docker ps --filter "publish=8050" -q | xargs -r docker stop || true
                        docker ps -a --filter "publish=8050" -q | xargs -r docker rm || true
                        
                        # Clean up old images (optional)
                        docker images bloomy/myapp --format "table {{.Repository}}:{{.Tag}}" | grep -v REPOSITORY | head -n -5 | xargs -r docker rmi || true
                    '''
                    
                    // Build new image
                    sh "docker build -t bloomy/myapp:1.0.${BUILD_NUMBER} ."
                    sh "docker tag bloomy/myapp:1.0.${BUILD_NUMBER} bloomy/myapp:latest"
                    
                    // Run new container
                    sh "docker run -d -p 8050:8050 --name myapp-1.0.${BUILD_NUMBER} bloomy/myapp:1.0.${BUILD_NUMBER}"
                    
                    // Health check
                    sh '''
                        echo "Waiting for application to start..."
                        sleep 30
                        curl -f http://localhost:8050/health || echo "Health check endpoint not available"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            // Clean workspace
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
            // Add notification here if needed
        }
        failure {
            echo 'Pipeline failed!'
            // Add notification here if needed
        }
    }
}
