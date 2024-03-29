pipeline {
    agent {
      kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: maven
            image: maven:3.9.5
            command:
            - cat
            tty: true
          - name: docker
            image: docker:20.10.0-git
            command:
            - cat
            tty: true
            volumeMounts:
             - mountPath: /var/run/docker.sock
               name: docker-sock
          - name: trivy
            image: aquasec/trivy
            command:
            - cat
            tty: true
            volumeMounts:
             - mountPath: "/root/.cache"
               name: trivy-db
          volumes:
          - name: docker-sock
            hostPath:
              path: /var/run/docker.sock
          - name: trivy-db
            hostPath:
              path: "/root/.cache"
        '''
      }
    }

    environment {
        DOCKERHUB_REGISTRY = 'innnnnwoo/springboot_test'
        DOCKERHUB_CREDENTIALS = credentials('docker_access_token')
        REVISION = '2.0'
    }

    stages {
		stage('Git Clone') {
            steps {
                container('maven') {
                    git branch: 'master',
                    changelog: false,
                    poll: false,
                    url: 'https://github.com/Da-SaNi/demo_springboot.git'
                }
            }
		}

        stage('Maven Build & SonarQube Analysis') {
            steps {
                container('maven') {
                    withSonarQubeEnv('sonarqube-server') {
                        sh 'mvn clean package sonar:sonar'
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Login Docker Hub') {
            steps {
                container('docker') {
                    script {
                        sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh 'docker build -t $DOCKERHUB_REGISTRY:$REVISION .'
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('docker') {
                    sh 'docker push $DOCKERHUB_REGISTRY:$REVISION'
                }
            }
        }

        stage('Security Analysis Docker Image using trivy') {
            steps {
                container('trivy') {
                    sh 'trivy image $DOCKERHUB_REGISTRY:$REVISION --format template --template "@/contrib/html.tpl" -o trivy_$REVISION.html'
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy_*', onlyIfSuccessful: true
        }
    }
}