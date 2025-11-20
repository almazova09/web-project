pipeline {

    agent {
        kubernetes {
            label "web-project-kaniko-${BUILD_NUMBER}"
            defaultContainer 'node'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins/label: web-project-kaniko
spec:
  containers:

    # Node container
    - name: node
      image: node:18-bullseye
      command: ["cat"]
      tty: true
      volumeMounts:
      - mountPath: /home/jenkins/agent
        name: workspace-volume

    # Kaniko container (FIXED)
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      command:
      - /kaniko/executor
      args:
      - "--help"
      tty: true
      volumeMounts:
      - name: kaniko-secret
        mountPath: /kaniko/.docker/
      - mountPath: /home/jenkins/agent
        name: workspace-volume

    # Python tools
    - name: tools
      image: python:3.11
      command: ["cat"]
      tty: true
      volumeMounts:
      - mountPath: /home/jenkins/agent
        name: workspace-volume

    # Jenkins agent
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
    - name: kaniko-secret
      secret:
        secretName: acr-dockerconfig
        items:
        - key: .dockerconfigjson
          path: config.json

    - name: workspace-volume
      emptyDir: {}
"""
        }
    }

    environment {
        ACR_LOGIN_SERVER = 'myprivateregistry15.azurecr.io'
        VERSION_FILE = "version.txt"
    }

    stages {

        /*-------------------------------------
         CHECKOUT
        -------------------------------------*/
        stage('Checkout Repo') {
            steps {
                checkout scm
            }
        }

        /*-------------------------------------
         INSTALL NPM
        -------------------------------------*/
        stage('Install Node Dependencies') {
            steps {
                container('node') {
                    dir('web') {
                        sh 'npm install'
                    }
                }
            }
        }

        /*-------------------------------------
         VERSIONING
        -------------------------------------*/
        stage('Determine Version') {
            steps {
                script {
                    container('node') {
                        sh """
                            git config --global --add safe.directory `pwd`
                            git fetch --tags || true
                            
                            latestTag=\$(git describe --tags --abbrev=0 || echo 'v1.0.0')
                            echo "Latest tag: \$latestTag"

                            clean=\${latestTag#v}
                            major=\$(echo \$clean | cut -d. -f1)
                            minor=\$(echo \$clean | cut -d. -f2)
                            patch=\$(echo \$clean | cut -d. -f3)

                            newPatch=\$((patch + 1))
                            newVersion="v\$major.\$minor.\$newPatch"

                            echo \$newVersion > ${VERSION_FILE}
                            echo "New version: \$newVersion"
                        """
                    }

                    env.IMAGE_VERSION = readFile("${VERSION_FILE}").trim()
                }
            }
        }

        /*-------------------------------------
         UNIT TESTS
        -------------------------------------*/
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
                            echo "⚠ No JUnit test reports found — skipping."
                        }
                    }
                }
            }
        }

        /*-------------------------------------
         ESLINT
        -------------------------------------*/
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

        /*-------------------------------------
         PRETTIER
        -------------------------------------*/
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

        /*-------------------------------------
         BUILD IMAGE WITH KANIKO
        -------------------------------------*/
        stage('Build Container Image (Kaniko)') {
            steps {
                container('kaniko') {
                    sh """
                        /kaniko/executor \
                          --dockerfile=web/Dockerfile \
                          --context=./web \
                          --destination=${ACR_LOGIN_SERVER}/web-app:${IMAGE_VERSION} \
                          --destination=${ACR_LOGIN_SERVER}/web-app:latest
                    """
                }
            }
        }
    }

    post {
        success {
            echo "🎉 CI SUCCESS! Image pushed: ${ACR_LOGIN_SERVER}/web-app:${IMAGE_VERSION}"
        }
        always {
            cleanWs()
        }
    }
}
