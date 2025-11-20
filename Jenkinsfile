pipeline {

    agent {
        kubernetes {
            label "web-project-kaniko-${BUILD_NUMBER}"
            defaultContainer 'node'
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
      - name: workspace-volume
        mountPath: /home/jenkins/agent

    - name: tools
      image: python:3.11
      command: ['cat']
      tty: true
      volumeMounts:
      - name: workspace-volume
        mountPath: /home/jenkins/agent

    # --- Kaniko container ---
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      command:
      - /busybox/cat
      tty: true
      volumeMounts:
      - name: workspace-volume
        mountPath: /home/jenkins/agent
      - name: kaniko-secret
        mountPath: /kaniko/.docker

  volumes:
    - name: workspace-volume
      emptyDir: {}
    - name: kaniko-secret
      secret:
        secretName: acr-dockerconfig  # MUST MATCH YOUR SECRET NAME
"""
        }
    }

    environment {
        ACR_LOGIN_SERVER = "myprivateregistry15.azurecr.io"
    }

    stages {

        /* ==========================
           1. CHECKOUT
        ========================== */
        stage("Checkout") {
            steps {
                checkout scm
            }
        }

        /* ==========================
           2. VERSIONING
        ========================== */
        stage("Set Version") {
            steps {
                script {
                    def tag = sh(script: "git describe --tags --abbrev=0 || echo v1.0.0", returnStdout: true).trim()
                    def clean = tag.replace("v","")
                    def (major, minor, patch) = clean.tokenize('.')
                    def newPatch = (patch as int) + 1
                    IMAGE_VERSION = "v${major}.${minor}.${newPatch}"

                    writeFile file: "version.txt", text: IMAGE_VERSION
                    echo "Using IMAGE_VERSION = ${IMAGE_VERSION}"
                }
            }
        }

        /* ==========================
           3. NPM INSTALL
        ========================== */
        stage("Install Node Dependencies") {
            steps {
                container('node') {
                    dir('web') {
                        sh 'npm install'
                    }
                }
            }
        }

        /* ==========================
           4. TESTS
        ========================== */
        stage("Run Unit Tests") {
            steps {
                container('node') {
                    dir("web") {
                        sh 'npm test --if-present || true'
                    }
                }
            }
        }

        /* ==========================
           5. ESLint
        ========================== */
        stage("ESLint") {
            steps {
                container('node') {
                    dir("web") {
                        sh '''if [ -f "eslint.config.js" ] || [ -f ".eslintrc.js" ]; then
                                npx eslint . || true
                              fi'''
                    }
                }
            }
        }

        /* ==========================
           6. Prettier
        ========================== */
        stage("Prettier") {
            steps {
                container('node') {
                    dir("web") {
                        sh '''if [ -f "prettier.config.js" ]; then
                                npx prettier --check . || true
                              fi'''
                    }
                }
            }
        }

        /* ==========================
           7. NPM AUDIT
        ========================== */
        stage("NPM Audit") {
            steps {
                container('node') {
                    dir('web') {
                        sh 'npm audit --audit-level=moderate || true'
                    }
                }
            }
        }

        /* ==========================
           8. SAST (njsscan)
        ========================== */
        stage("SAST (njsscan)") {
            steps {
                container('tools') {
                    sh '''
                        pip install njsscan || true
                        njsscan --json --output njsscan-results.json web || true
                    '''
                }
            }
        }

        /* ==========================
           9. KANIKO BUILD + PUSH
        ========================== */
        stage("Build & Push with Kaniko") {
            steps {
                container('kaniko') {
                    dir('web') {
                        sh """
                            /kaniko/executor \
                              --context `pwd` \
                              --dockerfile Dockerfile \
                              --destination=${ACR_LOGIN_SERVER}/web-app:${IMAGE_VERSION} \
                              --destination=${ACR_LOGIN_SERVER}/web-app:latest \
                              --skip-tls-verify
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "🎉 Build & Push SUCCESS | Version: ${IMAGE_VERSION}"
        }
    }
}
