pipeline {
  agent {
    kubernetes {
      label "spring-petclinic-${myid}"
      instanceCap 1
      defaultContainer 'jnlp'
      yaml '''
apiVersion: v1
kind: Pod
metadata:
  namespace: jenkins-workers
spec:
  # Use service account that can deploy to all namespaces
  serviceAccountName: default
  containers:
  - name: maven
    image: maven:3.8.1-openjdk-16
    command:
    - cat
    tty: true
    volumeMounts:
      - mountPath: "/root/.m2"
        name: m2
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.6.0-debug
    imagePullPolicy: Always
    command:
    - sleep
    args:
    - 99d    
    volumeMounts:
    - mountPath: "/root/.m2"
      name: m2      
    - name: docker-config
      mountPath: /kaniko/.docker
    - name: ca-cert
      mountPath: /kaniko/ssl/certs/
  volumes:
    - name: ca-cert
      secret:
        secretName: ca-bundle
        items:
        - key: additional-ca-cert-bundle.crt
          path: additional-ca-cert-bundle.crt
    - name: docker-config
      configMap:
        name: docker-config
    - name: m2
      persistentVolumeClaim:
        claimName: m2
'''
    }

  }
  stages {
    stage('Build') {
      steps {
        container(name: 'maven') {
          echo sh(script: 'env|sort', returnStdout: true)
          sh """
                                mvn -B -ntp -T 2 package -DskipTests -DAPP_VERSION=${APP_VER}
                                """
        }

      }
    }

    stage('Test') {
      parallel {
        stage(' Unit/Integration Tests') {
          post {
            always {
              archiveArtifacts(artifacts: 'target/**/*.jar', fingerprint: true)
              junit 'target/surefire-reports/**/*.xml'
            }

          }
          steps {
            container(name: 'maven') {
              sh """
                                            mvn -B -ntp -T 2 test -DAPP_VERSION=${APP_VER}
                                          """
            }

            jacoco(execPattern: 'target/*.exec', classPattern: 'target/classes', sourcePattern: 'src/main/java', exclusionPattern: 'src/test*')
          }
        }

        stage('Static Code Analysis') {
          steps {
            container(name: 'maven') {
              withSonarQubeEnv('My SonarQube') {
                sh """
                                                mvn sonar:sonar \
                                                  -Dsonar.projectKey=spring-petclinic \
                                                  -Dsonar.host.url=${env.SONAR_HOST_URL} \
                                                  -Dsonar.login=${env.SONAR_AUTH_TOKEN}
                                                """
              }

            }

          }
        }

      }
    }

    stage('Containerize') {
      steps {
        container(name: 'kaniko') {
          sh "sed -i 's,harbor.example.com,${env.HARBOR_URL},g' Dockerfile"
          sh "/kaniko/executor --dockerfile Dockerfile --context `pwd` --skip-tls-verify --destination=${env.HARBOR_URL}/library/samples/spring-petclinic:v1.0.${env.BUILD_ID}"
        }

      }
    }

    stage('Approval') {
      input {
        message 'Proceed to deploy?'
        id 'YES'
      }
      steps {
        echo 'Update helm chart to trigger GitOps-based deployment...'
      }
    }

    stage('GitOps-based Deploy') {
      steps {
        container(name: 'maven') {
          sh """
                                git config --global user.name $env.GIT_AUTHOR_NAME
                                git config --global user.email $env.GIT_AUTHOR_EMAIL
                                git clone https://$env.DEPLOY_GITREPO_USER:$env.DEPLOY_GITREPO_TOKEN@$env.DEPLOY_GITREPO_URL --branch=$env.DEPLOY_GITREPO_BRANCH deploy
                                # After cloning
                                cd deploy
                                # update values.yaml
                                sed -i -r 's,repository: (.+),repository: ${env.HARBOR_URL}/library/samples/spring-petclinic,' values.yaml
                                sed -i 's/tag: v1.0.*/tag: v1.0.${env.BUILD_ID}/' values.yaml
                                cat values.yaml
                                git commit -am 'bump up version number'
                                git remote set-url origin https://$env.DEPLOY_GITREPO_USER:$env.DEPLOY_GITREPO_TOKEN@$env.DEPLOY_GITREPO_URL
                                git push origin main
                              """
        }

      }
    }

  }
  environment {
    APP_VER = "v1.0.${BUILD_ID}"
    DEPLOY_GITREPO_USER = 'mbartosek'
    DEPLOY_GITREPO_URL = "github.com/${DEPLOY_GITREPO_USER}/spring-petclinic-helmchart.git"
    DEPLOY_GITREPO_BRANCH = 'main'
    DEPLOY_GITREPO_TOKEN = credentials('my-github')
  }
}