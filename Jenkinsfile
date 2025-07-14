pipeline {
    agent any

    stages {
        stage('Compile Code') {
            steps {
                sh '/opt/maven/bin/mvn compile'
            }
        }

        stage('PMD Code Review') {
            steps {
                sh '/opt/maven/bin/mvn -P metrics pmd:pmd'
            }
            post {
                success {
                    recordIssues(tools: [pmdParser(pattern: '**/pmd.xml')])
                }
            }
        }

        stage('Package Application') {
            steps {
                sh '/opt/maven/bin/mvn package'
            }
        }
    }
}
