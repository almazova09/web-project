pipeline {
    agent any

    environment {
        // Azure Container Registry
        ACR_LOGIN_SERVER = credentials('acr-login-server')
        ACR_USERNAME     = credentials('acr-username')
        ACR_PASSWORD     = credentials('acr-password')

        // SonarQube
        SONARQUBE_ENV = credentials('sonarqube-creds') // contains SONAR_TOKEN
        SONAR_HOST     = "http://4.242.72.128:9000"    // your LB

        // Semantic Version
        VERSION_FILE = "version.txt"
    }

    stages {

        // -------------------------------
        // 1. Checkout
        // -------------------------------
        stage('Checkout Repo') {
            steps {
                checkout scm
            }
        }

        // -------------------------------
        // 2. Install Dependencies
        // -------------------------------
        stage('Install Node Dependencies') {
            steps {
                dir('web') {
                    sh 'npm install'
                }
            }
        }

        // -------------------------------
        // 3. Semantic Versioning
        // -------------------------------
        stage('Determine Version') {
            steps {
                script {
                    def version = sh(
                        script: """
                        git fetch --tags
                        latestTag=\$(git describe --tags --abbrev=0 || echo 'v1.0.0')
                        echo "Latest tag: \$latestTag"

                        # Extract numbers
                        IFS='.' read -r major minor patch <<< "\${latestTag#v}"

                        # For PRs or new commits: increment patch
                        newPatch=\$((patch + 1))
                        newVersion="v\$major.\$minor.\$newPatch"

                        echo \$newVersion > ${VERSION_FILE}
                        echo "New version: \$newVersion"
                        """,
                        returnStdout: true
                    ).trim()

                    env.IMAGE_VERSION = readFile("${VERSION_FILE}").trim()
                }
            }
        }

        // -------------------------------
        // 4. Unit Tests
        // -------------------------------
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

        // -------------------------------
        // 5. ESLint
        // -------------------------------
        stage('Lint (ESLint)') {
            steps {
                dir('web') {
                    sh '''
                    if [ -f "eslint.config.js" ] || [ -f ".eslintrc.js" ]; then
                        npx eslint . || true
                    fi
                    '''
                }
            }
        }

        // -------------------------------
        // 6. Prettier check
        // -------------------------------
        stage('Code Style Check (Prettier)') {
            steps {
                dir('web') {
                    sh '''
                    if [ -f "prettier.config.js" ]; then
                        npx prettier --check . || true
                    fi
                    '''
                }
            }
        }

        // -------------------------------
        // 7. NPM Security Audit
        // -------------------------------
        stage('Dependency Vulnerability Check') {
            steps {
                dir('web') {
                    sh 'npm audit --audit-level=moderate || true'
                }
            }
        }

        // -------------------------------
        // 8. SAST Scan (njsscan)
        // -------------------------------
        stage('SAST Security Scan (NodeJS)') {
            steps {
                sh '''
                pip install njsscan || true
                njsscan --json --output njsscan-results.json web || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'njsscan-results.json', fingerprint: true
                }
            }
        }

        // -------------------------------
        // 9. API Schema Lint (if exists)
        // -------------------------------
        stage('API Spec Linting') {
            steps {
                sh '''
                if ls web/**/*api*.yml >/dev/null 2>&1; then
                    npm install -g @redocly/cli
                    redocly lint web/**/*.yml || true
                fi
                '''
            }
        }

        // -------------------------------
        // 10. SonarQube Scan
        // -------------------------------
        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('MySonarQube') {
                    dir('web') {
                        sh """
                        sonar-scanner \
                          -Dsonar.projectKey=web \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=${SONAR_HOST} \
                          -Dsonar.login=$SONARQUBE_ENV \
                          -Dsonar.projectVersion=${IMAGE_VERSION} \
                          -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                          -Dsonar.exclusions=node_modules/**,**/*.test.js
                        """
                    }
                }
            }
        }

        // -------------------------------
        // 11. Wait for SonarQube Quality Gate
        // -------------------------------
        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        // -------------------------------
        // 12. Docker Build
        // -------------------------------
        stage('Build Docker Image') {
            steps {
                dir('web') {
                    sh """
                    docker build -t web-app:${IMAGE_VERSION} -f Dockerfile .
                    """
                }
            }
        }

        // -------------------------------
        // 13. Trivy Scan
        // -------------------------------
        stage('Container Image Security Scan (Trivy)') {
            steps {
                sh """
                trivy image --severity CRITICAL,HIGH --exit-code 0 --no-progress web-app:${IMAGE_VERSION}
                """
            }
        }

        // -------------------------------
        // 14. Push to Azure Container Registry
        // -------------------------------
        stage('Push Docker Image to ACR') {
            steps {
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

    post {
        success {
            echo "CI pipeline completed successfully! Version: ${IMAGE_VERSION}"
        }
        always {
            cleanWs()
        }
    }
}
