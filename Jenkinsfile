pipeline {
    // No overall agent, rather, set up agent per stage; gives us decoupled stages
    agent none
   
    environment{
        IMAGETAG_VERSIONED="corstijank/blog-dotnet-jenkins:2.0-${env.BUILD_NUMBER}"
        IMAGETAG_LATEST="corstijank/blog-dotnet-jenkins:latest"
    }
    
    stages{
        stage('Build binaries'){
            // Run this stage in a docker container with the dotnet sdk
            agent { docker 'microsoft/dotnet:1.1.0-sdk-projectjson'}
            steps{
                // Unable to use 'dir' directive when running steps inside a docker container agent. Known issue at jenkins.
                git url: 'https://github.com/corstijank/blog-dotnet-jenkins.git'
                sh 'cd TodoApi && dotnet restore'
                sh 'cd TodoApi.Test && dotnet restore'
                sh 'cd TodoApi.Test && dotnet test -xml xunit-results.xml'
                sh 'cd TodoApi && dotnet publish project.json -c Release -r ubuntu.14.04-x64 -o publish'
                stash includes: 'TodoApi/publish/**', name: 'prod_bins' 
            }
            post{
                always{
                    step([$class    : 'XUnitBuilder',
                            thresholds: [[$class: 'FailedThreshold', failedThreshold: '1']],
                            tools     : [[$class: 'XUnitDotNetTestType', pattern: '**/xunit-results.xml']]])
                }
            }
        }
        stage('Create docker image'){
            // Run this stage any agent with a 'hasDocker' label
            agent { label 'hasDocker' }
            environment {
                DOCKER_ID = credentials('docker-id')
            }
            steps{
                // Unstash the binaries from the previous tage
                dir('TodoApi'){
                    unstash 'prod_bins'
                    sh """  docker build -t ${IMAGETAG_VERSIONED} .
                            docker tag ${IMAGETAG_VERSIONED} ${IMAGETAG_LATEST}
                            docker login -u ${DOCKER_ID_USR} -p ${DOCKER_ID_PSW}
                            docker push ${IMAGETAG_VERSIONED}
                            docker push ${IMAGETAG_LATEST} """
                }
            }
        }
        stage('Run in production'){
            agent { label 'hasDocker' }
            steps{
                
                dir('Environments/Production'){
                    sshagent(['corstijank-ssh']){
                        sh """ sed -ie 's#corstijank/blog-dotnet-jenkins:.*#${IMAGETAG_VERSIONED}#g' docker-compose.yml
                            docker-compose up -d
                            git config user.email "jenkins@staticsmustdie.net"
                            git config user.name "Jenkins"
                            git checkout master
                            git commit -am "updated to ${IMAGETAG_VERSIONED}"
                            git push """
                    }
                }
            }
        }
    }
}