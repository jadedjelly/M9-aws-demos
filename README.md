## Demo Project: 
### Complete the CI/CD Pipeline (Docker-Compose, Dynamic versioning)

#### Technologies used:
- AWS, Jenkins, Docker, Linux, Git, Java, Maven, Docker Hub

#### Project Description:
1. CI step: Increment version
2. CI step: Build artifact for Java Maven application
3. CI step: Build and push Docker image to Docker Hub
4. Improvement: Extract multiple Linux commands that are executed on remote server into a separate shell script and execute the script from Jenkinsfile
5. CD step: Deploy new application version with Docker Compose
6. CD step: Commit the version update

![M9image01.png](assets/M9image01.png)

- In the Jenkinsfile, we add the following from the previous module (Jenkins)
  - increment version (Before we build the app!)
  - commit version update (at the very end, so it updates the repos)

*NOTE: This is a perfect example of a Real World Pipeline!*
- There might be additional jobs, like notifying the team, updating a board, etc 

As below:

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

    environment {
        IMAGE_NAME = 'jadedjelly/mod9demo-app:java-maven-1.0'
    }
    stages {
        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
        }

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

        stage("deploy") {
            steps {
                script {
                    echo 'deploying docker image to EC2...'

                    def shell.mds = "bash ./server-.mds.sh ${IMAGE_NAME}"
                    def ec2Server = "ec2-user@18.202.21.18"
                    sshagent(['ec2-server-key']) {
                        sh "scp server-.mds.sh ${ec2Server}:/home/ec2-user"
                        sh "scp docker-compose.yaml ${ec2Server}:/home/ec2-user"
                        sh "ssh -o StrictHostKeyChecking=no ${ec2Server} ${shell.mds}"
                    }
                }
            }
        }
        }
        stage('commit version update'){
            steps {
                script {
                    echo "Increment version below commented out"
                    withCredentials([usernamePassword(credentialsId: 'github-creds', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh 'git remote set-url origin https://$USER:$PASS@github.com/jadedjelly/java-maven-app.git'
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump"'
                        sh 'git push origin HEAD:jenkins-jobs'
                    }
                  }
            }
        }
    }
```
- We could potentially move the new parts to the shared library!

*Note2:*
Nana points out that each stage of a pipeline should build on the next 
bump ver # > build the app with bumped vers > build app with bumped vers > deploy > commit new vers to pom file with current version 


