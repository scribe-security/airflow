pipeline {
    triggers {
        githubPush()
    }
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:dind
    command:
    - sh
    - -c
    - |
      dockerd-entrypoint.sh --host=tcp://127.0.0.1:2375
    tty: true
    securityContext:
      privileged: true
  - name: valint
    image: scribesecuriy.jfrog.io/scribe-docker-public-local/valint:latest 
    command:
    - cat
    tty: true
  - name: git
    image: alpine/git
    command:
      - cat
    tty: true
'''
        }
    }

    environment {
        IMAGE = "scribedocker/airflow:2.7.0"
    }

    stages {
        stage('checkout-bom') {
          steps {
            container('git') {
              withCredentials([string(credentialsId: 'github-access-token', variable: 'GITHUB_TOKEN')]) {
                sh 'git clone -b main --single-branch https://\$GITHUB_TOKEN@github.com/scribe-security/airflow.git'
              }
            }
            container('valint') {
              withCredentials([usernamePassword(credentialsId: 'scribe-research-auth-id', usernameVariable: 'SCRIBE_CLIENT_ID', passwordVariable: 'SCRIBE_CLIENT_SECRET')]) {
                sh '''
                valint bom dir:airflow \
                --context-type jenkins \
                --output-directory ./scribe/valint \
                -E -U $SCRIBE_CLIENT_ID -P $SCRIBE_CLIENT_SECRET \
                --product-key="airflow-jenkins" \
                --product-version="2.7.0" \
                --scribe.auth.audience=api.research.scribesecurity.com \
                --scribe.login-url=https://scribe-hub-dev.us.auth0.com \
                --scribe.url=https://airflow.research.scribesecurity.com '''
              }
            }
          }
        }

        stage('Build Airflow Image') {
            steps {
                container('docker') {
                    script {
                      withCredentials([usernamePassword(credentialsId: 'docker-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD docker.io"
                        dir('airflow') {
                          app = docker.build(env.IMAGE, "-f Dockerfile .")
                        }
                        sh '''
                        docker images
                        docker push scribedocker/airflow:2.7.0
                        '''
                      }
                    }
                }
            }
        }

        stage('Build Image BOM') {
            steps {
                container('valint') {
                    withCredentials([usernamePassword(credentialsId: 'scribe-research-auth-id', usernameVariable: 'SCRIBE_CLIENT_ID', passwordVariable: 'SCRIBE_CLIENT_SECRET')]) {
                        sh '''
                        valint bom scribedocker/airflow:2.7.0 \
                        --context-type jenkins \
                        --output-directory ./scribe/valint \
                        -E -U $SCRIBE_CLIENT_ID -P $SCRIBE_CLIENT_SECRET \
                        --product-key="airflow-jenkins" \
                        --product-version="2.7.0" \
                        --scribe.auth.audience=api.research.scribesecurity.com \
                        --scribe.login-url=https://scribe-hub-dev.us.auth0.com \
                        --scribe.url=https://airflow.research.scribesecurity.com '''
                    }
                }
            }
        }
    }
}
