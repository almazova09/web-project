pipeline {

    agent {
        kubernetes {
            label "web-project-${BUILD_NUMBER}"
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins/label: web-project
spec:
  containers:

    - name: node
      image: node:18-bullseye
      command: ['bash', '-c', 'sleep infinity']
      tty: true
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

    - name: tools
      image: python:3.11
      command: ['bash', '-c', 'sleep infinity']
      tty: true
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

    - name: azure-cli
      image: mcr.microsoft.com/azure-cli:latest
      command: ['bash', '-c', 'sleep infinity']
      tty: true
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

    - name: jnlp
      image: jenkins/inbound-agent:latest
      env:
        - name: JENKINS_AGENT_WORKDIR
          value: /home/jenkins/agent
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

  restartPolicy: Never

  volumes:
    - name: workspace-volume
      emptyDir: {}
"""
        }
    }

    environment {
        ACR_NAME = "myprivateregistry15"
        ACR_LOGIN_SERVER = "myprivateregistry15.azurecr.io"
        VERSION_FILE = "version.txt"
    }

    stages {

        stage('Checkout Repo') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                container('node') {
                    dir('web') {
                        sh 'npm install'
                    }
                }
            }
        }

        stage('Versioning') {
            steps {
                script {
                    container('node') {
                        sh '''
                            git config --global --add safe.directory $(pwd)
                            git fetch --tags || true

                            latest=$(git describe --tags --abbrev=0 || echo "v1.0.0")
                            clean=${latest#v}
                            major=$(echo $clean | cut -d. -f1)
                            minor=$(echo $clean | cut -d. -f2)
                            patch=$(echo $clean | cut -d. -f3)

                            newPatch=$((patch + 1))
                            echo "v${major}.${minor}.${newPatch}" > version.txt
                        '''
                    }
                    env.IMAGE_VERSION = readFile('version.txt').trim()
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
                always {
                    script {
                        def xml = findFiles(glob: 'web/test-results/**/*.xml')
                        if (xml.length > 0) {
                            junit 'web/test-results/**/*.xml'
                        } else {
                            echo "⚠ No JUnit reports found — skipping."
                        }
                    }
                }
            }
        }

        stage('ESLint') {
            steps {
                container('node') {
                    dir('web') {
                        sh '''
                            if [ -f ".eslintrc.js" ] || [ -f "eslint.config.js" ]; then
                                npx eslint . || true
                            fi
                        '''
                    }
                }
            }
        }

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

        stage('Build & Push (ACR Task Build)') {
            steps {
                container('azure-cli') {
                    withCredentials([azureServicePrincipal('AZURE_CREDENTIALS')]) {
                        sh '''
                            az login --service-principal \
                              -u $AZURE_CLIENT_ID \
                              -p $AZURE_CLIENT_SECRET \
                              --tenant $AZURE_TENANT_ID

                            az acr login --name ${ACR_NAME}

                            az acr build \
                              --registry ${ACR_NAME} \
                              --image web-app:${IMAGE_VERSION} \
                              --image web-app:latest \
                              --file web/Dockerfile \
                              web
                        '''
                    }
                }
            }
        }
    }

post {
    success {
        echo "🎉 CI completed successfully! Version: ${IMAGE_VERSION}"
    }
    always {
        cleanWs()
    }
}
}



