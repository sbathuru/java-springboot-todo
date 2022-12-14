pipeline {
   agent { label 'aws-jenkins-slave' }     
   //agent any
    triggers {
          pollSCM('4 4 4 * *')
    }
    environment {
          VER_NUM = "1.0.${BUILD_NUMBER}";
          REL_NUM = "1.0.${BUILD_NUMBER}.RELEASE";
          mavenHome =  tool name: "maven", type: "maven"
          namespace = "default"
     }
    tools{
          maven 'maven'
     }

    stages {
           stage ('Git Checkout') {
                 steps {
                     git credentialsId: 'github_credentials' , 
                         url: 'https://github.com/sbathuru/java-springboot-todo.git',  
                         branch: 'master'   
                  }
           }
           stage ('Maven Build') {
                        steps {
                            //sh "${mavenHome}/bin/mvn clean versions:set -Dver=${VER_NUM} package "
                            sh "${mavenHome}/bin/mvn clean test package verify"
                       }
           }
           stage ('Test & Jacoco Analysis') {
                        steps {
                            junit '**/target/surefire-reports/TEST-*.xml'
                            jacoco()
                       }
           }
           stage ('SonarQube Analysis') {
                 steps {
                      withSonarQubeEnv('sonar_server') {
                         sh "${mavenHome}/bin/mvn sonar:sonar -Dsonar.projectKey=java-springboot-todo"
                      }
                }
          }
          stage ('Artifactory configuration') {
            steps {
                rtServer (
                    id: "jfrog_server",
                    url: "https://sbathuru.jfrog.io/artifactory",
                    credentialsId: "jfrog_cred"
                )
                rtMavenDeployer (
                    id: "MAVEN_DEPLOYER",
                    serverId: "jfrog_server",
                    releaseRepo: "simpleapp-libs-release-local",
                    snapshotRepo: "simpleapp-libs-snapshot-local"
                )
                rtMavenResolver (
                    id: "MAVEN_RESOLVER",
                    serverId: "jfrog_server",
                    releaseRepo: "default-maven-virtual",
                    snapshotRepo: "default-maven-virtual"
                )
                rtMavenRun (
                    tool: "maven", // Tool name from Jenkins configuration
                    pom: 'pom.xml',
                    goals: 'clean install',
                    deployerId: "MAVEN_DEPLOYER",
                    resolverId: "MAVEN_RESOLVER"
                )
                rtPublishBuildInfo (
                    serverId: "jfrog_server"
             )
             }
            }
          stage('Docker Build & Push') {    
            steps {
               script{        // To add Scripted Pipeline sentences into a Declarative
                 try{
                     sh "pwd"
                 }catch(error){
                 }
               }
              sh "docker ps"
              withCredentials([string(credentialsId: 'dockerHubPwd', variable: 'dockerpwd')]) {
                sh "docker login -u sbathuru -p ${dockerpwd}"
              }
              sh "docker build -t sbathuru/java-springboot-todo:latest ."
              sh "docker tag sbathuru/java-springboot-todo:latest  sbathuru/java-springboot-todo:${VER_NUM}"
              sh "docker push sbathuru/java-springboot-todo:${VER_NUM}" 
              sh "docker push sbathuru/java-springboot-todo:latest" 
              sh "docker rmi sbathuru/java-springboot-todo" 
           } 
           }
          }
/*    
        stage('Deploy Into DEV (Docker)') {
           steps {   
               sh "pwd"
               sshagent(['aws-private-key-mumbai']) {
                    sh "ssh -o StrictHostKeyChecking=no ec2-user@docker.bathuru.shop  sudo docker rm -f java-springboot-todo|| true"
                    sh "ssh -o StrictHostKeyChecking=no ec2-user@docker.bathuru.shop  sudo docker run  -d -p 80:8080 --name java-springboot-todo sbathuru/java-springboot-todo:latest"
                }
            }
         }

        stage('Deploy Into PROD (K8S)') {
           steps {   
               //sh "kubectl apply -f simpleapp-deploy-k8s.yaml"
               kubernetesDeploy(
                configs: 'simpleapp-deploy-k8s.yaml',
                kubeconfigId: 'k8s_cluster_kubeconfig',
                enableConfigSubstitution: true
                )
            }
         }
         */
/*
         stage('Build Helm Charts') {
            steps {
              dir('charts') {
                sh "helm package simpleapp"
					      sh "helm push-artifactory --username sbathuru --password Sridevi@116   simpleapp-helm-0.0.3.tgz https://sbathuru.jfrog.io/artifactory/simpleapp-helm-local"
					  }
          }
        } 
     	  stage ('Deploy Into PROD - Helm Charts')  {
	      steps {
           sh "helm version"
           dir("charts/simpleapp"){
             sshagent(['aws-ap-south-pem']) {
                    sh 'pwd'
                    sh "scp -o StrictHostKeyChecking=no Chart.yaml  ec2-user@k8s-bootstrap.bathur.xyz:/home/ec2-user/"
                    sh "ssh -o StrictHostKeyChecking=no ec2-user@k8s-bootstrap.bathur.xyz   helm repo add simpleapp-helm  https://sbathuru.jfrog.io/artifactory/simpleapp-helm-local --username sbathuru  --password Sridevi@116"
                    sh "ssh -o StrictHostKeyChecking=no ec2-user@k8s-bootstrap.bathur.xyz   helm repo update"
                    sh "ssh -o StrictHostKeyChecking=no ec2-user@k8s-bootstrap.bathur.xyz   helm upgrade simpleapp-helm  --install  --force ."
                }
                }
        }
      }
*/
    post {
       success { 
                echo 'Pipeline Sucessfully Finished' 
                slackSend (
                  message: "Build SUCESS !!! - JOB : ${env.JOB_NAME} BUILD_NO ${env.BUILD_NUMBER} More Info at ${env.BUILD_URL} "
                )
                SendEmailNotification("Sucess")
               }
       failure { 
               echo 'Pipeline Failure' 
               slackSend (
                  message: "Build FAILURE !!! - JOB : ${env.JOB_NAME} BUILD_NO ${env.BUILD_NUMBER} More Info at ${env.BUILD_URL} "
                )
                SendEmailNotification("Failed")
              }
       always {
              echo 'From Always!!' 
               }
          }
}

def SendEmailNotification(String result) {
     def bcontent =""
     if(result == "Sucess") {
       bcontent = "Your project got Build and Deployed successfully !!!"
     } else { 
       bcontent =  "Your project got Failed during Build and Deployement !!!"
     }
      mail bcc: '', 
      body: """Hi Team, 
      ${bcontent}

      Please find the details as below,
      Job Name: ${env.JOB_NAME}
      Job URL : ${env.JOB_URL}
      Build URL: ${env.BUILD_URL}

      Thanks
      DevOps Team""", 
      cc: '', 
      from: '', 
      replyTo: '', 
      subject: "${result}!!! - ${env.JOB_NAME} - Build # ${env.BUILD_NUMBER}", 
      to: 'bathuru.srinivas@gmail.com'
  }