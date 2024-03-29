pipeline {
     environment {
       IMAGE_NAME = "serviio"
       IMAGE_TAG = "latest"
       STAGING = "serviio-staging"
       PRODUCTION = "serviio-production"
       PRODUCTION_IP_HOST = "<IP_PUBLIC>"
     }
     agent none
     stages {
         stage('Build image') {
             agent any
             steps {
                script {
                  sh 'docker build -t cfreijanes/$IMAGE_NAME:$IMAGE_TAG .'
                }
             }
        }
        stage('Run container based on builded image') {
            agent any
            steps {
                 script {
                 sh '''
                   docker stop $IMAGE_NAME || true
                   docker rm -f $IMAGE_NAME || true
                   docker run --name $IMAGE_NAME -d -p 23423:23423 -p 8081:8081 -p 1900:1900 -v /etc/localtime:/etc/localtime:ro cfreijanes/$IMAGE_NAME:$IMAGE_TAG
                   sleep 180
                 '''
               }
            }
       }
       stage('Test image') {
           agent any
           steps {
              script {
                sh '''
                   curl http://localhost:23423/console/#/app/welcome/ | grep "serviio"
                '''
              }
           }
       }
       stage('Push image on Dockerhub') {
          agent any
          environment {
              DOCKERHUB_CREDENTIALS = credentials('dockerhub_cfreijanes')
          }
          steps {
             script {
               sh '''
                 echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                 docker push cfreijanes/$IMAGE_NAME:$IMAGE_TAG
               '''
              }
          }
       } 
       stage('Clean Container') {
          agent any
          steps {
             script {
               sh '''
                 docker stop $IMAGE_NAME
                 docker rm -f $IMAGE_NAME
               '''
             }
          }
     }
     stage('Push image in production and deploy it') {
       when {
              expression { GIT_BRANCH == 'origin/master' }
            }
      agent any
      steps {
           withCredentials([sshUserPrivateKey(credentialsId: "private_key", keyFileVariable: 'keyfile', usernameVariable: 'NUSER')]) 
           {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') 
            {                        
                 script
             {                           
                  timeout(time: 15, unit: "MINUTES") 
              {                                
                   input message: 'Do you want to approve the deploy in production?', ok: 'Yes'                            
              }
            sh '''
              ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${PRODUCTION_IP_HOST} docker stop $IMAGE_NAME  || true
              ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${PRODUCTION_IP_HOST} docker rm $IMAGE_NAME  || true
              ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${PRODUCTION_IP_HOST} docker rmi cfreijanes/$IMAGE_NAME:$IMAGE_TAG  || true
              ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${PRODUCTION_IP_HOST} docker run --name $IMAGE_NAME -d -p 23423:23423 -p 8081:8081 -p 1900:1900 -v /etc/localtime:/etc/localtime:ro cfreijanes/$IMAGE_NAME:$IMAGE_TAG  || true
            '''
             }
           }
        }
     }
    }
          stage('Deploy app on EC2-cloud alpine') {
            agent {
                docker {
                    image('alpine')
                    args ' -u root'
                }
            }
            when{
                expression{ GIT_BRANCH == 'origin/master' }
            }
            steps{
                withCredentials([sshUserPrivateKey(credentialsId: "private_key", keyFileVariable: 'keyfile', usernameVariable: 'NUSER')]) {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        script{
                            timeout(time: 15, unit: "MINUTES") {
                                input message: 'Do you want to approve the deploy in production?', ok: 'Yes'
                            }						
                            sh'''
                              apk update
                              which ssh-agent || ( apk add openssh-client )
                              eval $(ssh-agent -s)
                              ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${PRODUCTION_IP_HOST} docker stop $IMAGE_NAME  || true
                              ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${PRODUCTION_IP_HOST} docker rm $IMAGE_NAME  || true
                              ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${PRODUCTION_IP_HOST} docker rmi cfreijanes/$IMAGE_NAME:$IMAGE_TAG  || true
                              ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${PRODUCTION_IP_HOST} docker run --name $IMAGE_NAME -d -p 23423:23423 -p 8081:8081 -p 1900:1900 -v /etc/localtime:/etc/localtime:ro cfreijanes/$IMAGE_NAME:$IMAGE_TAG  || true
                            '''
                        }
                    }
                }
            }
        }
    }
            post {
             success {
               slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
               }
             failure {
               slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
               }  
 }
}
