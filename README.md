## Description


Project on End to End CI/CD pipeline for java based application using Git,Github,Jenkins,Maven,Sonarqube,Nexus,Slack,Docker and Kuberenets with ECR as private docker registry and Zero Downtime Deployment.


![end-to-end-cicd-FLOW](https://user-images.githubusercontent.com/96073033/152660142-349279ac-e756-4d2c-abc8-acca3f12d958.JPG)

----

## Project Flow

```sh
1. Developer pushes code into Github.
2. Webhook triggers Jenkins when there is a change in the code
3. Jenkins Pulls the Code from the Github
4. Maven builds the code and generates artifacts
5. Code quality is measured with Sonarqube
6. Quality Gate Check , If Quality Gate Fails Jenkins Job will fail !!!!!! (Triggered by Sonarqube Webhooks)
7. Upload artifact generated into Sonatype Nexus . It will dynamically choose Snapshot or release repository based on the version tag in pom.xml
8. Build Docker Image based on the Dockerfile with projectname && commit-id as tag . So each time it will be different.
9. Push Docker Image to private ECR docker registry.
10.Dynamically change image in pod template in manifest file.
11.Deploy to K8s cluster created with kubeadm . It will pull image from private registry.
12. Send Build Notification over Slack Channel and email notification when build is success/failure.

Note: We can add an approval step before deploying to K8s cluster as an input from user.

```
----

## Features
```sh
1. Zero downtime deployment with rolling update as deployment strategy
2. Complete automation as when developer check in code , deployed to k8s cluster
3. Versioning of docker images , build artifacts.
4. Code checked against Code coverage and whether coding stantards are met.

```
----

## Pipeline Execution

![JENKINS-PIPELINE-VIEW-MAIN](https://user-images.githubusercontent.com/96073033/152660815-81535f95-a123-48d9-a4dc-73d2849a9fde.JPG)

## Execution Results

SONARQUBE REPORTS

![SONAR REPORT](https://user-images.githubusercontent.com/96073033/152660873-3f392176-1387-46f5-8d92-4f35011eb6d1.JPG)

NEXUS UPLOADING

![NEXUS REPOSITORY](https://user-images.githubusercontent.com/96073033/152660904-bc77d9a6-5e3e-43ab-9e31-e09d6a16b20d.JPG)

ECR 

![ECR REPO](https://user-images.githubusercontent.com/96073033/152660928-0e337f03-4e69-45cb-99ea-9b4b4d8a0e8f.JPG)

JOB TRIGGERED BY WEBHOOKS

![GITHUB-WEBHOOK-PUSH](https://user-images.githubusercontent.com/96073033/152660992-0ac37278-5d83-4c70-ad9d-2048490464c4.JPG)

K8s CLUSTER

```sh
Controller --> Deployment
Strategy --> Rolling Update
```

![K8S cluster](https://user-images.githubusercontent.com/96073033/152660993-5f416527-ad3b-4b4d-9b76-59aa68b2804d.JPG)

SLACK NOTIFICATION

![SLACK - NOTIFICATION](https://user-images.githubusercontent.com/96073033/152661219-32260921-02b7-44d7-aa2c-8966f5e55114.JPG)

EMAIL NOTIFICATION

![EMAIL NOTIFICATION](https://user-images.githubusercontent.com/96073033/152661223-51e1bcf7-5a73-4f55-8f6e-5e3cc1995ae7.JPG)

FINAL RESULT

![FINAL UPDATE](https://user-images.githubusercontent.com/96073033/152661000-a29889f0-0f6f-4a60-8375-7c477fa9f95c.JPG)



## Jenkins Pipeline

Change account number and credentials as per your environment.

```sh

def COMMIT
def BRANCH_NAME
def GIT_BRANCH
pipeline
{
 agent any
 environment
 {
     AWS_ACCOUNT_ID="<Your account Id>"
     AWS_DEFAULT_REGION="us-east-1" 
     IMAGE_REPO_NAME="mavenwebapp"
     REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
     
 }
 tools
 {
      maven 'MAVEN_3.8.4'
 }   

 options 
 {
  buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '4', daysToKeepStr: '', numToKeepStr: '4')
  timestamps()
}
 stages
 {
     stage('Code checkout')
     {
         steps
         {
             script
             {
                 checkout([$class: 'GitSCM', branches: [[name: '*/development']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/NITHIN-JOHN-GEORGE/JAVA-CI-CD-PIPELINE.git']]])
                 COMMIT = sh (script: "git rev-parse --short=10 HEAD", returnStdout: true).trim()  
                 
                 
                 

             }
             
         }
     }
     stage('Build')
     {
         steps
         {
             sh "mvn clean package"
         }
     }
     stage('Execute Sonarqube Report')
     {
         steps
         {
            withSonarQubeEnv('Sonarqube-Server') 
             {
                sh "mvn sonar:sonar"
             }  
         }
     }
     stage('Quality Gate Check')
     {
         steps
         {
             timeout(time: 1, unit: 'HOURS') 
             {
                waitForQualityGate abortPipeline: true, credentialsId: 'SONARQUBE-CRED'
            }
         }
     }
     
     stage('Nexus Upload')
     {
         steps
         {
             script
             {
                 def readPom = readMavenPom file: 'pom.xml'
                 def nexusrepo = readPom.version.endsWith("SNAPSHOT") ? "wallmart-snapshot" : "wallmart-release"
                 nexusArtifactUploader artifacts: 
                 [
                     [
                         artifactId: "${readPom.artifactId}",
                         classifier: '', 
                         file: "target/${readPom.artifactId}-${readPom.version}.war", 
                         type: 'war'
                     ]
                ], 
                         credentialsId: 'Nexus-Cred', 
                         groupId: "${readPom.groupId}", 
                         nexusUrl: '3.82.213.203:8081', 
                         nexusVersion: 'nexus3', 
                         protocol: 'http', 
                         repository: "${nexusrepo}", 
                         version: "${readPom.version}"

             }
         }
     }
     stage('Login to AWS ECR')
     {
         steps
         {
             script
             {
                 sh "/usr/local/bin/aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
             }
         }
     }
     stage('Building Docker Image')
     {
         steps
         {
             script
             {
              sh "docker build . -t ${REPOSITORY_URI}:mavenwebapp-${COMMIT}"
             }
         }
     }
     stage('Pushing Docker image into ECR')
     {
         steps
         {
             script
             {
                 sh "docker push ${REPOSITORY_URI}:mavenwebapp-${COMMIT}"
             }
         }

     }
     stage('Update image in K8s manifest file')
     {
         steps
         {
             
                 sh """#!/bin/bash
                 sed -i 's/VERSION/$COMMIT/g' deployment.yaml
                 """
             }
         }
     
     stage('Deploy to K8s cluster')
     {
         steps
         {
             sh 'kubectl apply -f deployment.yaml --record=true'
             sh """#!/bin/bash
             sed -i 's/$COMMIT/VERSION/g' deployment.yaml
             """

         }
     }
 }

 post
 {
     always
     {
         cleanWs()
     }
     success
     {
        slackSend channel: 'build-notifications',color: 'good', message: "started  JOB : ${env.JOB_NAME}  with BUILD NUMBER : ${env.BUILD_NUMBER}  BUILD_STATUS: - ${currentBuild.currentResult} To view the dashboard (<${env.BUILD_URL}|Open>)"
        emailext attachLog: true, body: '''BUILD IS SUCCESSFULL - $PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS:
 
        Check console output at $BUILD_URL to view the results.
 
        Regards,
 
        Nithin John George
        ''', compressLog: true, replyTo: '<Your mail id>', 
        subject: '$PROJECT_NAME - $BUILD_NUMBER - $BUILD_STATUS', to: 'njdevops321@gmail.com'
     }
     failure
     {
         slackSend channel: 'build-notifications',color: 'danger', message: "started  JOB : ${env.JOB_NAME}  with BUILD NUMBER : ${env.BUILD_NUMBER}  BUILD_STATUS: - ${currentBuild.currentResult} To view the dashboard (<${env.BUILD_URL}|Open>)"
         emailext attachLog: true, body: '''BUILD IS FAILED - $PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS:
 
        Check console output at $BUILD_URL to view the results.
 
        Regards,
 
        Nithin John George
        ''', compressLog: true, replyTo: '<Your mail id>', 
        subject: '$PROJECT_NAME - $BUILD_NUMBER - $BUILD_STATUS', to: '<Your mail id>'
     }
 }

}

```

## K8s manifest File (Deployment)

```sh

apiVersion: apps/v1
kind: Deployment
metadata:
  name: mavenwebapp-dp
  labels:
    app: mavenwebapp
spec:
  replicas: 4
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  minReadySeconds: 30
  selector:
    matchLabels:
       app: mavenwebapp
  template:
    metadata:
      name: mavenwebapp-pod
      labels:
        app: mavenwebapp
    spec:
      imagePullSecrets:
      - name: regcrd
      containers:
      - name: mavenwebapp-container
        image: <AWS_ACCOUNT_NUMBER>.dkr.ecr.us-east-1.amazonaws.com/mavenwebapp:mavenwebapp-VERSION
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 300m
            memory: 256Mi
          limits:
            cpu: 800m
            memory: 1Gi
 
---

apiVersion: v1
kind: Service
metadata:
  name: mavenwebapp-svc
spec:
  type: NodePort
  selector:
    app: mavenwebapp
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30003
```

## Docker File

```sh

FROM tomcat:8.0.20-jre8
 
COPY target/java-web-app*.war /usr/local/tomcat/webapps/java-web-app.war

```
