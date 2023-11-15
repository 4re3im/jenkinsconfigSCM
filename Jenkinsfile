pipeline {
    agent none
    
    options {
        skipDefaultCheckout()
    }

    stages {
        stage('Build Package') {
            agent {
                label "cloud-agent-1"
            }  
            steps {
                script {

                    //create jenkinsconfig directory              
                    sh '''
                    mkdir -p "${WORKSPACE}/jenkinsconfig"
                    '''

                    //checkout jenkinsconfig
                    dir("${WORKSPACE}/jenkinsconfig") {
                        sshagent(['sshgithub']) {
                            git branch: 'main', credentialsId: 'sshgithub', url: 'git@github.com:4re3im/jenkinsconfig.git'
                        }
                    }

                    //extract branch and repo url value
                    def branch = sh(script: "cat ${WORKSPACE}/jenkinsconfig/Branch.txt | grep '^branch=' | cut -d'=' -f2", returnStdout: true).trim()
                    def repo_url = sh(script: "cat ${WORKSPACE}/jenkinsconfig/Branch.txt | grep '^giturl=' | cut -d'=' -f2", returnStdout: true).trim()
                   
                    env.BRANCH = branch
                    env.REPO_URL = repo_url
                    echo "BRANCH: ${env.BRANCH}"
                    echo "REPO_URL: ${env.REPO_URL}"

                    //checkout repo url
                    sshagent(['sshgithub']) {
                        git branch: env.BRANCH, credentialsId: 'sshgithub', url: env.REPO_URL
                    }

                    //extract branch and package name
                    def version = sh(script: "cat ${WORKSPACE}/version.txt | grep '^version=' | cut -d'=' -f2", returnStdout: true).trim()
                    def packageName = sh(script: "cat ${WORKSPACE}/version.txt | grep '^package=' | cut -d'=' -f2", returnStdout: true).trim()

                    env.VERSION = version
                    env.PACKAGE_NAME = packageName
                    echo "Version: ${env.VERSION}"
                    echo "Package: ${env.PACKAGE_NAME}"

                    //Create RPM package
                    sh '''
                    #!/bash/bin

                    # Enable error-sensitive shell
                    set -e

                    if [ -d SOURCES ]; then
                        rm -r SOURCES
                        mkdir SOURCES
                    fi
                    cp -rp $WORKSPACE/SOURCE/. SOURCES
                    find SOURCES -name .git | sed -e 's/^://' -e 's/$//' | xargs rm -rf

                    if [ -d SPECS ]; then
                        rm -r SPECS
                        mkdir SPECS
                    fi
                    cp -rp $WORKSPACE/SPEC/. SPECS
                    find SPECS -name .git | sed -e 's/^://' -e 's/$//' | xargs rm -rf
                    '''

                    sh '''
                    #!/bin/bash

                    # Enable error-sensitive shell
                    set -e

                    rpmbuild --define "_version ${VERSION}" --define "_release $BUILD_NUMBER" --define "_topdir $WORKSPACE" -bb SPECS/${PACKAGE_NAME}.spec
                    '''
                    
                    //Transfer RPM to td repo
                    withCredentials([sshUserPrivateKey(credentialsId: 'ec2-user', keyFileVariable: 'SSH_KEY')]) {
                    def remoteIp = '34.243.35.123'
                    sh """
                    ssh -o StrictHostKeyChecking=no -l ec2-user -i \${SSH_KEY} $remoteIp 'whoami'
                    scp -i \${SSH_KEY} $WORKSPACE/RPMS/noarch/${PACKAGE_NAME}-${VERSION}-${BUILD_NUMBER}.noarch.rpm ec2-user@$remoteIp:/home/ec2-user/
                     """
                    }

                    
                    // Push to S3
                    /*withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY', credentialsId: 'AWS-BONP']]) {
                        sh "aws s3 cp $WORKSPACE/RPMS/noarch/${PACKAGE_NAME}-${VERSION}-${BUILD_NUMBER}.noarch.rpm s3://bnr-jenkins/package-repository/${PACKAGE_NAME}-${VERSION}-${BUILD_NUMBER}.noarch.rpm --region eu-west-1"
                    }*/
                }
            }
        }
        stage ('Deploy to Staging') {
            agent {
                label 'built-in'
            }
            steps {
                input message: 'Approval to Deploy to staging', submitter: 'Dom AWS Admins'
            }
        }
        /*stage ('RPM') {
            parallel {
                stage ('Downloading RPM from S3') {
                    agent {
                        label 'cloud-agent-1'
                    }
                    steps {
                        script {
                            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY', credentialsId: 'AWS-BONP']]) {
                            sh "aws s3 cp s3://bnr-jenkins/package-repository/${PACKAGE_NAME}-${VERSION}-${BUILD_NUMBER}.noarch.rpm $WORKSPACE/RPMS/noarch/ --region eu-west-1"    
                            }
                        }
                        
                    } 
            
                }
                stage ('Donwloading RPM for transfer') {
                    agent {
                        label 'built-in'
                    }
                    steps {
                        echo 'Transfering RPM to other server'
                    }
                }
            }
        }*/
    }
    post {
        always {
            emailext attachLog: true, body: '$DEFAULT_CONTENT', subject: '$DEFAULT_SUBJECT', to: '$DEFAULT_RECIPIENTS'
        }
    }
}
