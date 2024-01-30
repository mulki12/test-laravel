def INSTANCE_IP="10.1.63.14"
def INSTANCE_USER="jenkins"
def AWS_ACCOUNT_ID="221047265242"
def AWS_DEFAULT_REGION="ap-southeast-1"
def NAMESPACE="cd-jenkins"
def NAME_APP="test-laravel"
def IMAGE_TAG=""
def CODE_REPO="https://github.com/mulki12/test-laravel.git"
def CREDENTIAL_CODE_REPO="github-mulki"
def REPO_CONFIG="https://github.com/mulki12/test-laravel-config.git"
def CREDENTIAL_CONFIG_REPO="github-mulki"
def KUBECONFIG="config"
def REPOSITORY_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/test-laravel"

pipeline {

  agent {
    kubernetes {
      defaultContainer "jnlp"
      yaml """
        apiVersion: v1
        kind: Pod
        metadata:
          labels:
            component: ci
        spec:
          tolerations:
          - key: "jenkins"
            operator: "Equal"
            value: "agent"
            effect: "NoSchedule"
          affinity:
            nodeAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:
               - preference:
                   matchExpressions:
                   - key: jenkins
                     operator: In
                     values:
                     - agent
                 weight: 100
          # Use service account that can deploy to all namespaces
          serviceAccountName: cd-jenkins
          containers:
          - name: helm
            image: masfikri/aws-helm-kubectl:v3
            imagePullPolicy: IfNotPresent
            command:
            - cat
            tty: true
          - name: jnlp
            image: masfikri/jnlp-agent:4.13
            imagePullPolicy: IfNotPresent
      """
    }
  } 

  stages {

    stage('clone') {
            steps {
                container('jnlp') {
                    checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'refs/heads/master']],
                    //branches: [[name: params.BRANCH]],
                    extensions: [[
                        $class: 'RelativeTargetDirectory',
                        relativeTargetDir: 'code']],
                    userRemoteConfigs: [[
                        url: "${CODE_REPO}",
                        credentialsId: "${CREDENTIAL_CODE_REPO}",
                    ]]
                ])

                    checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'refs/heads/master']],
                    extensions: [[
                        $class: 'RelativeTargetDirectory',
                        relativeTargetDir: 'repo-config']],
                    userRemoteConfigs: [[
                        url: "${REPO_CONFIG}",
                        credentialsId: "${CREDENTIAL_CONFIG_REPO}",
                    ]]
                ])
                }
            }
        }

    stage("build image") {
      environment {
        REPOSITORY_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/test-laravel"
      }
      steps {
        container("jnlp") {
            dir('code') {
               script {

                    sh 'git rev-parse --short HEAD > .git/commit-id'

                    def commit_id = readFile('.git/commit-id').trim()
                    IMAGE_TAG = commit_id.substring(0,7)

                    
                    sh "ls -lah"
                    sh "pwd"
                    sshagent(["pem-credential"]) {

                    sh "whoami"
                    //sh "scp -o StrictHostKeyChecking=no rm -rf /home/jenkins/agent/workspace/${NAME_APP}/.git"
                    //sh "ls -lah /home/jenkins/agent/workspace/${NAME_APP}/.git/objects"
                    //sh "rm -rf /home/jenkins/agent/workspace/${NAME_APP}/.git/objects"
                    //sh "rsync --recursive --exclude=/home/jenkins/agent/workspace/test-laravel/config/.git/objects"
                    sh "scp -o StrictHostKeyChecking=no -r ../code ${INSTANCE_USER}@${INSTANCE_IP}:/home/${INSTANCE_USER}/agent/workspace/"

                    
                    sh "ssh -o StrictHostKeyChecking=no ${INSTANCE_USER}@${INSTANCE_IP} docker build -t ${REPOSITORY_URI}:${IMAGE_TAG} /home/${INSTANCE_USER}/agent/workspace/${NAME_APP}/code"

                    sh "ssh -o StrictHostKeyChecking=no ${INSTANCE_USER}@${INSTANCE_IP} docker push ${REPOSITORY_URI}:${IMAGE_TAG}"

                    }



                    sh "ls -lah"
               }
            }
        }
      }
    }

    stage('hello AWS') {
            steps {
                withAWS(credentials: 'aws-cred', region: 'ap-southeast-1') {
                    //sh 'echo "hello KB">hello.txt'
                    //s3Upload acl: 'Private', bucket: 'test-laravel', file: 'hello.txt'
                    //s3Download bucket: 'kb-bucket', file: 'downloadedHello.txt', path: 'hello.txt'
                    //sh 'cat downloadedHello.txt'
                }
            }
        }
      

    stage('deployment') {
      environment {
        REPOSITORY_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/test-laravel"
      }
        steps{
          container('helm'){
            script {
              withAWS(credentials: 'aws-cred', region: 'ap-southeast-1') {
                dir ('repo-config') {
                  echo "Deploy to cluster ${KUBECONFIG}"
                  sh "mkdir -p /root/.kube/"
                  //sh "whoami"
                  //sh "touch /root/.kube/config"
                  //writeFile file: '/root/.kube/config', text: readFile(KUBECONFIG)
                  //sh 'sudo chmod u+x /usr/local/bin/kubectl', text:readFile(KUBECONFIG)
                  sh "aws ecr get-login-password --region ap-southeast-1 | aws eks update-kubeconfig --name EKS-Cluster --region ap-southeast-1"
                  //sh "pwd"
                  //sh "cp -f ../.kube* /root/.kube/config"
                  sh "helm version"
                  sh "helm ls -A"
                  sh """
            helm upgrade ${NAME_APP} ./helm/${NAME_APP} \
            --set-string image.repository=${REPOSITORY_URI},image.tag=${BUILD_ID} \
            -f ./helm/values.dev.yml --debug --install --namespace ${NAMESPACE}
            """
                }
            }

          }
          
          
        }
    }
    }
    // stage('Deploying App to Kubernetes') {
    //   steps {
    //     withKubeConfig([credentialsId: 'config', serverUrl: '']) {
    //       //sh 'cat deploymentservice.yml | sed "s/{{BUILD_NUMBER}}/$BUILD_NUMBER/g" | kubectl apply -f -'
    //       sh 'sudo chmod u+x /usr/local/bin/kubectl'
    //       sh 'kubectl apply -f deploymentservice.yml'
    //     }
    //   }
    // }
    // Uploading Docker images into AWS ECR

    stage('Pushing to ECR') {
        steps{  
          script {
            sh 'aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin 221047265242.dkr.ecr.ap-southeast-1.amazonaws.com' 
            sh 'docker tag $JOB_NAME:$BUILD_ID ${REPOSITORY_URI}:$BUILD_ID'
            sh 'docker push ${REPOSITORY_URI}:$BUILD_ID'
            sh 'docker rmi $JOB_NAME:$BUILD_ID ${REPOSITORY_URI}:$BUILD_ID' // Delete docker images from server 
          }
        }
      }

//    stage('Deploy our image') {
//      steps{
//        script {
//          docker.withRegistry( '', registryCredential ) {
//            dockerImage.push("latest")
//          }
//        }
//      }
//    }

  }
}






