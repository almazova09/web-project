pipeline {

    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: node
      image: node:18-bullseye
      command: ['cat']
      tty: true
      volumeMounts:
        - name: dockersock
          mountPath: /var/run/docker.sock

    - name: docker
      image: docker:24.0
      command: ['cat']
      tty: true
      volumeMounts:
        - name: dockersock
          mountPath: /var/run/docker.sock

    - name: tools
      image: python:3.11
      command: ['cat']
      tty: true

  volumes:
    - name: dockersock
      hostPath:
        path: /var/run/docker.sock
"""
        }
    }

    environment {
        ACR_LOGIN_SERVER = 'myprivateregistry15.azurecr.io'
        SONAR_HOST = "http://4.242.72.128:9000"
        VERSION_FILE = "version.txt"
    }

    stages {

        stage('Checkout Repo') {
            steps { checkout scm }
        }

        stage('Install Node Dependencies') {
            steps {
                container('node') {
                    dir('web') {
                        sh 'npm install'
                    }
                }
            }
        }

        stage('Determine Version') {
            steps {
                script {
                    container('node') {
                        sh """
                            git fetch --tags || true
                            latestTag=\$(git describe --tags --abbrev=0 || echo 'v1.0.0')
                            IFS='.' read -r major minor patch <<< "\${latestTag#v}"
                            newPatch=\$((patch + 1))
                            newVersion="v\$major.\$minor.\$newPatch"
                            echo \$newVersion > ${VERSION_FILE}
                        """
                    }
                    env.IMAGE_VERSION = readFile("${VERSION_FILE}").trim()
                }
            }
        }

        stage('Run Unit Tests') {
            steps {
                container('node') {
                    dir('web') {
                        sh 'npm test --if-present || true'
                    }
                }
            }
            post {
                always { junit 'web/test-results/**/*.xml' }
            }
        }

        stage('ESLint') {
            steps {
                container('node') {
                    dir('web') {
                        sh 'npx eslint . || true'
                    }
                }
            }
        }

        stage('Prettier') {
            steps {
                container('node') {
                    dir('web') {
                        sh 'npx prettier --check . || true'
                    }
                }
            }
        }

        stage('NPM Audit') {
            steps {
                container('node') {
                    dir('web') {
                        sh 'npm audit --audit-level=moderate || true'
                    }
                }
            }
        }

        stage('SAST (njsscan)') {
            steps {
                container('tools') {
                    sh """
                        pip install njsscan || true
                        njsscan --json --output njsscan-results.json web || true
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'njsscan-results.json', fingerprint: true
                }
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('MySonarQube') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        container('node') {
                            dir('web') {
                                sh """
                                    sonar-scanner \
                                        -Dsonar.projectKey=web \
                                        -Dsonar.sources=. \
                                        -Dsonar.host.url=${SONAR_HOST} \
                                        -Dsonar.login=${SONAR_TOKEN} \
                                        -Dsonar.projectVersion=${IMAGE_VERSION} \
                                        -Dsonar.exclusions=node_modules/**
                                """
                            }
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
                container('docker') {
                    dir('web') {
                        sh "docker build -t web-app:${IMAGE_VERSION} -f Dockerfile ."
                    }
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                container('tools') {
                    sh """
                        pip install trivy || true
                        trivy image --severity CRITICAL,HIGH --exit-code 0 --no-progress web-app:${IMAGE_VERSION}
                    """
                }
            }
        }

        stage('Push Image to ACR') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'acr-credentials', usernameVariable: 'ACR_USERNAME', passwordVariable: 'ACR_PASSWORD')
                ]) {
                    container('docker') {
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
    }

    post {
        success { echo "CI pipeline completed successfully! Version: ${IMAGE_VERSION}" }
        always { cleanWs() }
    }
}
