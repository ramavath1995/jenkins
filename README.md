# jenkins
1. create Security group
    * jenkins
        22-myip
        8080-anywhere
        8080-sonarSG
    * nexus
        22-myip
        8081-myip
        8081-jenkinsSG
    * sonar
        22-myip
        80-myip
        80-sonarSG

2. Create instances with respective user data for
    1. jenkins(ubuntu22-(t2-medium)) ss -tulpn | grep 8080 
        * install plugins in jenkins
            Maven Integration
            GitHub Integration
            Nexus Artifact Uploader
            SonarQube Scanner
            Slack Notification
            Build Timestamp
    2. nexus(centos9-(t2-medium)) ss -tulpn | grep 8081
        * Create maven repositories
            release-hosted
            central-proxy(https://repo1.maven.org/maven2/)
            snapshot-hosted
            group-group
    3. sonar(ubuntu22-(t2-medium)) ss -tulpn | grep 9000
        * login sonar server
3. Create public key and copy the in github
    * ssh-keygen
    * cat ~/.ssh/id_rsa.pub
        ssh -T git@github.com   # to chech your laptop is connected to github
            Hi dhakya31! You've successfully authenticated, but GitHub does not provide shell access.(u 'll get this meaasge)
    * cat .git/config  (It gives repository information)
      cat ~/.gitconfig (It gives git user infomation)

4. Login to jnkins
    ssh into jenkins
    * Go to manage jenkins -> global tool configuration  and add
        i.jdk
            install jdk8
                apt install openjdk-8-jdk -y
                ls /usr/lib/jvm  (/usr/lib/jvm/java-1.8.0-openjdk-amd64 )
        ii.maven(3.6.3)
    * Add maven credentials in manage jenkins -> Credentials -> System -> Global credentials (unrestricted)

        * Select username with password -> username(nexus user name) -> password(nexus password) -> ID(name your choice in snake case) -> same as ID

1.Now write a Jenkinsfile
        Build stage syntax:
            pipeline{
                agent any
                tools{
                    maven 'maven3'      # Give these tools name which is given in global tool names
                    jdk 'jdk8'
                }

                environment{
                    NEXUS_USER = "admin"
                    NEXUS_PASS = "admin"
                    NEXUS_IP = "172.31.60.48"
                    RELEASE_REPO = "release"
                    SNAPSHOT_REPO  = "snapshot"
                    CENTRAL_REPO = "central"
                    GROUP_REPO = "group"
                    NEXUS_PORT = "8081"
                    NEXUS_LOGIN = "nexus_login"
                    SONARSCANNER = "sonarscanner"
                    SONARSERVER = "sonarserver"
                    AWS_CREDENTIALS = "awscreds"
                    ECR_LOCATION = ""
                    IMAGE_URL = ""

                }
                stages{
                    stage('Build'){
                        steps{
                            sh 'mvn -s settings.xml -DskipTests clean install'  # we are passing the settings and skiping the test we don't  need test at this stage
                        }
                    }
                }
            }
    Before commit we have to change the pom.xml and settings.xml
    * Change the name of the as given in jenkins file environment name settings.xml and pom.xml
    * Create a pipeline 
        Select pipeline from scm -> goto github and from code select ssh and copy the url and paste in jinkins -> add github credentials with ssh private key 
            * authenticate from private key for that 
            cat ~/.ssh/id_rsa
    note:ssh the  jenkins become the root user and the become jenkins user for that(su - jenkins  or su -s /bin/bash jenkins)
            run the command which shown in jenkins server for example shown below
                git ls-remote -h -- git@github.com:dhakya31/jenkins.git HEAD
                cat .ssh/known_hosts (to check jenkins user added)

    * Create webhook in github(http://jenkinspubip:8080/github-webhook)
2. Write a test stage
    stage('test'){
        steps{
            sh 'mvn test'
        }
    }

    * Test test stages runs the unit test and generates the reports later  use this reports to upload in sonar server
3. Write checkstyle stage
    stage('Check style'){
        steps{
            sh 'mvn checkstyle:checkstyle'
        }
    }
     * checkstyle is code analysis tool it checks the issues and vulnerability with code 
            Vulnerability:
                A vulnerability is a hole or a weakness in the application, which can be a design flaw or an implementation bug, that allows an attacker to cause harm to the stakeholders of an application.

4. i.Now we have check the code with sonar for that go to manage jenkins -> global tool configuration -> sonarQube scanner add the tool same as maven tool added (select sonarQube Scanner 4.7)
and then
ii. manage jenkins > configure system >  SonarQube servers > name it and added (http://sonar private ip) >
generate sonar token and save in jenkins for that
    => go to sonar server -> admistrator -> my account -> security -> generate token copy and add in jenkins same as github

    add sonarscanner and sonarserver details in environment in pipeline

    stage('Sonar Analysis'){
        environment{
            sonarHome = tool "${SONARSCANNER}"
        }
        steps{
            withSonarQubeEnv("${SONARSERVER}"){
                sh '''${sonarHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                -Dsonar.projectName=vprofile \
                -Dsonar.projectVersion=1.0 \
                -Dsonar.sources=src/ \
                -Dsonar.java.binaries= targe/test-classes/com/viswalpathit/account/controllerTest \
                -Dsonar.checkstyle.reportsPath= target/checkstyle-result.xml\
                -Dsonar.jacoco.reportsPath= target/jacoco.exec\
                -Dsonar.junit.reportsPath=target/surefire-reports \'''
            }
        }
    }
* create webhook on sonar server for respective project for quality gates
    (http://jenkins private ip:8080/sonarqube-webhook)  then
    Write a stage for quality gate
    
    stage("Quality Gate"){
        steps{
            timeout{time: 1, units: 'HOURS'}{
                waitForQualityGate abortPipeline: true
            }
            
        }
    }
    
* upload artifact to nexus repo 
      stage("Artifact Uploader"){
            steps{
                nexusArtifactUploader(
                nexusVersion: 'nexus3',
                protocol: 'http',
                nexusUrl: "${NEXUS_IP}:${NEXUS_PORT}",
                groupId: 'practice',
                version: "${env.BUILD_ID}",
                repository: "${RELEASE_REPO}",
                credentialsId: "${NEXUS_LOGIN}",
                artifacts: [
                    [artifactId: 'jenkinsci',
                    classifier: '',
                    file: 'target/vprofile-v2.war',
                    type: 'war']
                    ]
                )
            }
        }
    
5. For slack notifications install app create workspace, channel and generate token by adding jenkins-ci app in slack and add this token jenkins steps ar
    manage jenkins > system configuration >choose secrete text
This block is at top stage of pipeline
def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger'
]
And add post to same level as stages shown below 
   post{
        always{
            echo "Slack Notification"
            slackSend channel: "#devops",
            color: COLOR_MAP[currentBuild.currentResult],
            message: "*${currentBuild.currentResult}*: job '${env.JOB_NAME}' build ${env.BUILD_NUMBER} \n more info at: ${env.BUILD_URL}"
        }
    }


#######################################  This is up to jenkinsci   #########################################
    

    Jenkins cicd
    ============

    pre-requisites
    ---------------


    1. Create separate job for cicd along with same jenkins file 
    2. create ecr and  iam
        iam has following polices
            * AmazonEC2ContainerRegistryFullAccess
            * AmazonECS_FullAccess
    3. Install plugins in jenkins
        * CloudBees Docker Build and Publish
        * Amazon ECR
        * Docker Pipeline
        * Pipeline: AWS Steps
    4. Add aws credentials in jenkins (global credentials)

    5. Install docker in jenkin server
        * Add docker user  in jenkins group for
            su - jenkins
            id
            exit
            usermod -aG docker jenkins if want check user added or not run previous steps
    6. Copy the docker file
          # https://github.com/devopshydclub/vprofile-project.git
    write a stage for build a image continuation to  jenkins ci

        stage("Docker Build"){
            steps{
                script{
                    docker.build(appregistry + ":$bUILD_NUMBR", "./docker/Dockerfile")
                }
            }
        }
    and push the image to aws ecr
        stage("Push Image"){
            steps{
                script{
                    docker.withRegistry(awscreds, image_url){
                        dockerImage.push("latest")
                        dockerImage.push("$BUILD_NUMBER")
                    }
                }
            }
        }
        
# jenkinscicd
        stage('Upload Artifact to ECR'){
            steps{
                scricpt{
                    docker.withRegistry(AWS_CREDS, IMAGE_URL){
                        dockerImage.push('latest')
                        dockerImage.push(${BUILD_NUMBER})
                    }
                }
            }
    
