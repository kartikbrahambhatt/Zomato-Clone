pipeline {
    agent any

    tools {
        nodejs 'Node18'
    }

    environment {
        NODE_OPTIONS = '--openssl-legacy-provider'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Node Version') {
            steps {
                sh 'node -v'
                sh 'npm -v'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    npm config set audit false
                    npm config set fund false
                    npm install
                '''
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=zomato-clone \
                            -Dsonar.projectName=Zomato-Clone \
                            -Dsonar.sources=src
                        """
                    }
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    sh """
                        ${tool 'DependencyCheck'}/bin/dependency-check.sh \
                        --project Zomato-Clone \
                        --scan . \
                        --format XML \
                        --out . \
                        --nvdApiKey \$NVD_API_KEY
                    """
                }

                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }

        stage('Trivy File System Scan') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    sh '''
                        trivy fs \
                        --skip-version-check \
                        --severity HIGH,CRITICAL \
                        .
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t kartikkbhatt/zomato-clone:latest .'
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push kartikkbhatt/zomato-clone:latest
                        docker logout
                    '''
                }
            }
        }

    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed!'
        }
    }
}
