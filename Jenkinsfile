pipeline {
    agent any

    environment {
        // Azure Container Registry
        ACR_LOGIN_SERVER = 'myprivateregistry15.azurecr.io'   
        
        SONAR_HOST = "http://4.242.72.128:9000"
        VERSION_FILE = "version.txt"
    }

    stages {

        stage('Checkout Repo') {
            steps {
                checkout scm
            }
        }

        stage('Install Node Dependencies') {
            steps {
                dir('web') {
                    sh 'npm install'
                }
            }
        }

        stage('Determine Version') {
            steps {
                script {
                    sh """
                        git fetch --tags || true
                        latestTag=\$(git describe --tags --abbrev=0 || echo 'v1.0.0')

                        IFS='.' read -r major minor patch <<< "\${latestTag#v}"
                        newPatch=\$((patch + 1))
                        newVersion="v\$major.\$minor.\$newPatch"

                        echo \$newVersion > ${VERSION_FILE}
                    """

                    env.IMAGE_VERSION = readFile("${VERSION_FILE}").trim()
                }
            }
        }

        stage('Run Unit Tests') {
            steps {
                dir('web') {
                    sh 'npm test --if-present || true'
                }
            }
            post {
                always {
                    junit 'web/test-results/**/*.xml'
                }
            }
        }

        stage('Lint (ESLint)') {
            steps {
                dir('web') {
                    sh """
                        if [ -f "eslint.config.js" ] || [ -f ".eslintrc.js" ]; then
                            npx eslint . || true
                        fi
                    """
                }
            }
        }

        stage('Code Style Check (Prettier)') {
            steps {
                dir('web') {
                    sh """
                        if [ -f "prettier.config.js" ]; then
                            npx prettier --check . || true
                        fi
                    """
                }
            }
        }

        stage('Dependency Vulnerability Check') {
            steps {
                dir('web') {
                    sh 'npm audit --audit-level=moderate || true'
                }
            }
        }

        stage('SAST Security Scan (njsscan)') {
            steps {
                sh """
                    pip install njsscan || true
                    njsscan --json --output njsscan-results.json web || true
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'njsscan-results.json', fingerprint: true
                }
            }
        }

        stage('API Spec Linting') {
            steps {
                sh """
                    if ls web/**/*api*.yml >/dev/null 2>&1; then
                        npm install -g @redocly/cli
                        redocly lint web/**/*.yml || true
                    fi
                """
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('MySonarQube') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        dir('web') {
                            sh """
                                sonar-scanner \
                                    -Dsonar.projectKey=web \
                                    -Dsonar.sources=. \
                                    -Dsonar.host.url=${SONAR_HOST} \
                                    -Dsonar.login=${SONAR_TOKEN} \
                                    -Dsonar.projectVersion=${IMAGE_VERSION} \
                                    -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                                    -Dsonar.exclusions=node_modules/**,**/*.test.js
                            """
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('web') {
                    sh """
                        docker build -t web-app:${IMAGE_VERSION} -f Dockerfile .
                    """
                }
            }
        }

        stage('Container Image Security Scan (Trivy)') {
            steps {
                sh """
                    trivy image --severity CRITICAL,HIGH --exit-code 0 --no-progress web-app:${IMAGE_VERSION}
                """
            }
        }

        stage('Push Docker Image to ACR') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'acr-credentials', usernameVariable: 'ACR_USERNAME', passwordVariable: 'ACR_PASSWORD')
                ]) {
                    sh """
                        echo ${ACR_PASSWORD} | docker login ${ACR_LOGIN_SERVER} -u ${ACR_USERNAME} --password-stdin

                        docker tag web-app:${IMAGE_VERSION} ${ACR_LOGIN_SERVER}/web-app:${IMAGE_VERSION}
                        docker tag web-app:${IMAGE_VERSION} ${ACR_LOGIN_SERVER}/web-app:latest

                        docker push ${ACR_LOGIN_SERVER}/web-app:${IMAGE_VERSION}
                        docker push ${ACR_LOGIN_SERVER}/web-app:latest
                    """
                }
            }
        }
    }

    post {
        success {
            echo "CI completed successfully! Version: ${IMAGE_VERSION}"
        }
        always {
            cleanWs()
        }
    }
}
