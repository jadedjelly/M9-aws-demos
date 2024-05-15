## Demo Project: 
### CD - Deploy Application from Jenkins Pipeline to EC2 Instance (automatically with docker)

#### Technologies used:
- AWS, Jenkins, Docker, Linux, Git, Java, Maven, Docker Hub

#### Project Description:
1. Prepare AWS EC2 Instance for deployment (Install Docker)
2. Create ssh key credentials for EC2 server on Jenkins
3. Extend the previous CI pipeline with deploy step to ssh into the
remote EC2 instance and deploy newly built image from Jenkins
server
4. Configure security group on EC2 Instance to allow access to our
web application

1. was done in the previous demo

## Install SSH Agent plugin & create SSH Credential type

- Jenkins server > plugins > Available Plugins > search for ssh agent ([here](https://plugins.jenkins.io/ssh-agent/))

![M9image01.png](assets/M9image01.png)

- Create a new multibranch (MB) pipeline for this module

![M9image02.png](assets/M9image02.png)

- From the MB dashboard > click credentials > click the link under "scoped to Module 9 - AWS-Multibranch-pipeline" > click "global credentials (unrestricted)" > add credentials
  - From kind, select SSH username with Private key
  - name it: "ec2-server-key"
    - give the same name for description
  - user: "ec2-user"
  - click the button for Private key, and paste contents of pem file in here
  - Click Create

![M9image03.png](assets/M9image03.png)


## Jenkinsfile Syntax for a plugin

- Click into the MB pipeline, and there's an option on the left side called "Pipeline Syntax"
  - **This is a nice little way you can look up syntax rules for plugins without having to google**
- Use the dropdown menu, and select "sshagent: SSH agent"
  - This will list all the SSH creds we have, we only have the one so it's already selected (as usual there's an option to add)
  - You can then press "Generate Pipeline Script" and it'll display the script needed to be able to use the SSH creds

```groovy
sshagent(['ec2-server-key']) {
    // some block
}
``` 

- start-code branch, we edit the Jenkinsfile
  - add a shell command, with ssh and the public ip
  - we need to suppress the popup, cos its not in interactive mode, we add "-o StrictHostKeyChecking=no"
  - the docker run .md, needs to be added as a variable (I suspect because it's cleaner)


```groovy
stage("deploy") {
            steps {
                script {
                    def docker.md = 'docker run -d -p 3080:3080 jadedjelly/reactnodejsapp:1.0'
                    sshagent(['ec2-server-key']) {
                        sh ssh -o StrictHostKeyChecking=no ec2-user@18.202.21.18 ${docker.md}"
                    }
                }
            }
        }  
```

- we don't need to run docker login, as we did it already on the server
  - From the server, I have removed the container & image
- Commit the change
- On the EC2 instance we need to make a FW change so the Jenkins server can communicate with the Instance, see below for changes (also needed to edit the app, as we changed it to 3080)

![M9image04.png](assets/M9image04.png)

- we run the pipeline code and access it on the browser

Jenkins output console:

![M9image05.png](assets/M9image05.png)

EC2:
![M9image06.png](assets/M9image06.png)

Browser:
![M9image07.png](assets/M9image07.png)

## Executing complete pipelines

![M9image08.png](assets/M9image08.png)


- From the jenkins-jobs branch, we edit the Jenkinsfile with the correct IMAGE_NAME, EC2 ip & correct Jenkins Shared Library (and associated credentials)

```groovy
#!/usr/bin/env groovy

library identifier: 'jenkins-shared-library@main', retriever: modernSCM(
    [$class: 'GitSCMSource',
    remote: 'https://github.com/jadedjelly/jenkins-shared-library.git',
    credentialsID: 'github-creds'
    ]
)

pipeline {
    agent any
    tools {
        maven 'maven-3.9'
    }
    enviroment {
        IMAGE_NAME = 'jadedjelly/mod9demo-app:java-maven-1.0'
    }
    stages {
        stage('build app') {
            steps {
                echo 'building application jar...'
                buildJar()
            }
        }
        stage('build image') {
            steps {
                script {
                    echo 'building the docker image...'
                    buildImage(env.IMAGE_NAME)
                    dockerLogin()
                    dockerPush(env.IMAGE_NAME)
                }
            }
        }

        stage("deploy")
        steps {
            script {
                echo 'deploying docker image to EC2...'
                def docker.md = "docker run -p 8080:8080 -d ${IMAGE_NAME}"
                sshagent(['ec2-server-key']) {
                  sh "ssh -o StrictHostKeyChecking=no ec2-user@18.202.21.18 ${docker.md}"
                }
            }
        }
//        }
//        stage('commit version update'){
//            steps {
//                script {
//                    echo "Increment version below commented out"
//                    withCredentials([usernamePassword(credentialsId: 'github-creds', passwordVariable: 'PASS', usernameVariable: 'USER')]){
//                        sh 'git remote set-url origin https://$USER:$PASS@github.com/jadedjelly/java-maven-app.git'
//                        sh 'git add .'
//                        sh 'git commit -m "ci: version bump"'
//                        sh 'git push origin HEAD:jenkins-jobs'
//                    }
//                  }
//            }
//        }
//    }
    }
}

```

- for the "Deploy stage", we could extract that and add it to the shared library
- we change the ports to be 8080 (as this is the port java-maven is running on)
- commit the changes
- Open the 8080 port on the EC2 instance

- App deployed via pipeline, no issues that end, app running on EC2 no issues, but getting error in browser, waiting on Support team (06/04/2024),
  - no longer waiting on support team, understand concept, will build similar in personal projects 

![M9image09.png](assets/M9image09.png)
