pipeline {

    agent {
        kubernetes {
            label "web-project-ci-${BUILD_NUMBER}"
            defaultContainer 'node'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins/label: web-project-ci
spec:
  containers:
    - name: node
      image: node:18-bullseye
      command: ["cat"]
      tty: true
      volumeMounts:
      - mountPath: /home/jenkins/agent
        name: workspace-volume
      - mountPath: /var/run/docker.sock
        name: dockersock

    - name: docker
      image: docker:24.0
      command: ["cat"]
      tty: true
      volumeMounts:
      - mountPath: /var/run/docker.sock
        name: dockersock
      - mountPath: /home/jenkins/agent
        name: workspace-volume

    - name: tools
      image: python:3.11
      command: ["cat"]
      tty: true
      volumeMounts:
      - mountPath: /home/jenkins/agent
        name: workspace-volume

    - name: jnlp
      image: jenkins/inbound-agent:latest
      env:
      - name: JENKINS_AGENT_WORKDIR
        value: /home/jenkins/agent
      volumeMounts:
      - mountPath: /home/jenkins/agent
        name: workspace-volume

  restartPolicy: Never
  volumes:
    - name: dockersock
      hostPath:
        path: /var/run/docker.sock
    - name: workspace-volume
      emptyDir: {}
"""
        }
    }

    environment {
        ACR_LOGIN_SERVER = 'myprivateregistry15.azurecr.io'
        VERSION_FILE     = "version.txt"
        IMAGE_VERSION    = ""
    }

    stages {

        /*------------------------------
         CHECKOUT
        -------------------------------*/
        stage('Checkout Repo') {
            steps {
                checkout scm
            }
        }

        /*------------------------------
         SET VERSION
        -------------------------------*/
        stage('Set Version') {
            steps {
                script {
                    IMAGE_VERSION = "v1.${env.BUILD_NUMBER}"
                    writeFile file: VERSION_FILE, text: IMAGE_VERSION
                    echo "Using IMAGE_VERSION = ${IMAGE_VERSION}"
                }
            }
        }

        /*------------------------------
         NPM INSTALL
        -------------------------------*/
        stage('Install Node Dependencies') {
            steps {
                container('node') {
                    dir('web') {
                        sh 'npm install'
                    }
                }
            }
        }

        /*------------------------------
         UNIT TESTS
        -------------------------------*/
        stage('Run Unit Tests') {
            steps {
                container('node') {
                    dir('web') {
                        sh 'npm test --if-present || true'
                    }
                }
            }
            post {
                always {
                    script {
                        def files = findFiles(glob: 'web/test-results/**/*.xml')
                        if (files.length > 0) {
                            junit 'web/test-results/**/*.xml'
                        } else {
                            echo "⚠ No JUnit test reports found — skipping JUnit parsing."
                        }
                    }
                }
            }
        }

        /*------------------------------
         ESLINT
        -------------------------------*/
        stage('ESLint') {
            steps {
                container('node') {
                    dir('web') {
                        sh '''
                            if [ -f "eslint.config.js" ] || [ -f ".eslintrc.js" ]; then
                                npx eslint . || true
                            fi
                        '''
                    }
                }
            }
        }

        /*------------------------------
         PRETTIER
        -------------------------------*/
        stage('Prettier') {
            steps {
                container('node') {
                    dir('web') {
                        sh '''
                            if [ -f "prettier.config.js" ]; then
                                npx prettier --check . || true
                            fi
                        '''
                    }
                }
            }
        }

        /*------------------------------
         NPM AUDIT
        -------------------------------*/
        stage('NPM Audit') {
            steps {
                container('node') {
                    dir('web') {
                        sh 'npm audit --audit-level=moderate || true'
                    }
                }
            }
        }

        /*------------------------------
         SAST: NJSSCAN
        -------------------------------*/
        stage('SAST (njsscan)') {
            steps {
                container('tools') {
                    sh '''
                        pip install njsscan || true
                        njsscan --json --output njsscan-results.json web || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'njsscan-results.json', fingerprint: true
                }
            }
        }

        /*------------------------------
         DOCKER BUILD
        -------------------------------*/
        stage('Build Docker Image') {
            steps {
                container('docker') {
                    dir('web') {
                        sh """
                            docker build -t web-app:${IMAGE_VERSION} -f Dockerfile .
                        """
                    }
                }
            }
        }

        /*------------------------------
         TRIVY SCAN
        -------------------------------*/
        stage('Trivy Scan') {
            steps {
                container('docker') {
                    sh """
                        trivy image --severity CRITICAL,HIGH --exit-code 0 --no-progress web-app:${IMAGE_VERSION}
                    """
                }
            }
        }

        /*------------------------------
         PUSH IMAGE
        -------------------------------*/
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
        success {
            echo "CI completed successfully! Version: ${IMAGE_VERSION}"
        }
        always {
            cleanWs()
        }
    }
}
