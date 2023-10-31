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
                                  
                    sh '''
                    mkdir -p "${WORKSPACE}/jenkinsconfig"
                    '''
                    dir("${WORKSPACE}/jenkinsconfig") {
                        sshagent(['sshgithub']) {
                            git branch: 'main', credentialsId: 'sshgithub', url: 'git@github.com:4re3im/jenkinsconfig.git'
                        }
                    }

                    def branch = sh(script: "cat ${WORKSPACE}/jenkinsconfig/Branch.txt | grep '^branch=' | cut -d'=' -f2", returnStdout: true).trim()
                    def repo_url = sh(script: "cat ${WORKSPACE}/jenkinsconfig/Branch.txt | grep '^giturl=' | cut -d'=' -f2", returnStdout: true).trim()
                   
                    env.BRANCH = branch
                    env.REPO_URL = repo_url
                    
                    // Print the values
                    echo "BRANCH: ${env.BRANCH}"
                    echo "REPO_URL: ${env.REPO_URL}"


                    sshagent(['sshgithub']) {
                        git branch: env.BRANCH, credentialsId: 'sshgithub', url: env.REPO_URL
                    }

                    def version = sh(script: "cat ${WORKSPACE}/version.txt | grep '^version=' | cut -d'=' -f2", returnStdout: true).trim()
                    def packageName = sh(script: "cat ${WORKSPACE}/version.txt | grep '^package=' | cut -d'=' -f2", returnStdout: true).trim()

                    env.VERSION = version
                    env.PACKAGE_NAME = packageName

                    echo "Version: ${env.VERSION}"
                    echo "Package: ${env.PACKAGE_NAME}"

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
                    
                    // Push to S3
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY', credentialsId: 'AWS-BONP']]) {
                        sh "aws s3 cp $WORKSPACE/RPMS/noarch/${PACKAGE_NAME}-${VERSION}-${BUILD_NUMBER}.noarch.rpm s3://bnr-jenkins/package-repository/${PACKAGE_NAME}-${VERSION}-${BUILD_NUMBER}.noarch.rpm --region eu-west-1"
                    }
                }
            }
        }
        stage ('Copy RPM from S3') {
            agent {
                label 'built-in'
            }
            steps {
                input message: 'Approval to Copy from S3', submitter: 'Dom AWS Admins'
            }
        }
        stage ('Download RPM from S3') {
                agent {
                 label 'cloud-agent-1'
            }
            steps {
                script {
                    // Set your AWS credentials here if needed
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY', credentialsId: 'AWS-BONP']]) {
                    sh "aws s3 cp s3://bnr-jenkins/package-repository/${PACKAGE_NAME}-${VERSION}-${BUILD_NUMBER}.noarch.rpm $WORKSPACE/RPMS/noarch/ --region eu-west-1"
                    }
                }
            }
        }
    }
}
