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
          volumes:
          - name: docker-sock
            hostPath:
              path: /var/run/docker.sock
        '''
      }
    }

    environment {
        DOCKERHUB_REGISTRY = 'innnnnwoo/springboot_test'
        REVISION = '1.0'
        DOCKERHUB_CREDENTIALS = credentials('docker_access_token')
        SONARQUBE_TOKEN = credentials('sonarqube_token')
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

        stage('SonarQube Analysis') {
            steps {
                container('maven') {
                    sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=demo_springboot -Dsonar.host.url=http://sonarqube-svc.sonarqube.svc.cluster.local:9000 -Dsonar.login=$SONARQUBE_TOKEN'
                }
            }
        }

        stage('Build-Jar-file') {
            steps {
                container('maven') {
                    sh 'mvn package'
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


    }
}