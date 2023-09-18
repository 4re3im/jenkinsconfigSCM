pipeline {
    agent {
        label "built-in"
    }
    parameters {
        string(name: 'BRANCH', defaultValue: 'jenkins-branch-01', description: 'Branch')
        string(name: 'REPO_URL', defaultValue: 'git@github.com:4re3im/JenkinsTest.git', description: 'Repository URL')
    }

    options {
        skipDefaultCheckout()
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                script {
                    sshagent(['sshgit']) {
                        git branch: "${params.BRANCH}", credentialsId: 'sshgit', url: params.REPO_URL
                    }
                }
            }
        }

        stage('Read Version and Package') {
            steps {
                script {
                    def version = sh(script: "cat ${WORKSPACE}/version.txt | grep '^version=' | cut -d'=' -f2", returnStdout: true).trim()
                    def packageName = sh(script: "cat ${WORKSPACE}/version.txt | grep '^package=' | cut -d'=' -f2", returnStdout: true).trim()

                    env.VERSION = version
                    env.PACKAGE_NAME = packageName

                    echo "Version: ${env.VERSION}"
                    echo "Package: ${env.PACKAGE_NAME}"
                }
            }
        }

        stage('Prepare Files') {
            steps {
                script {
                    sh '''
                    #!/bin/bash

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
                }
            }
        }

        stage('Create RPM Package') {
            steps {
                script {
                    sh '''
                    #!/bin/bash

                    # Enable error-sensitive shell
                    set -e

                    rpmbuild --define "_version ${VERSION}" --define "_release $BUILD_NUMBER" --define "_topdir $WORKSPACE" -bb SPECS/${PACKAGE_NAME}.spec
                    '''
                }
            }
        }

        stage ('Approval to push to S3') {
            steps {
                input message: 'Approval', submitter: 'read'
            }
        }

        stage('Push to S3') {
            steps {
                script {
                    // Use AWS CLI to upload the ZIP file to S3 using stored credentials
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY', credentialsId: 'f022a85a-84f2-4cf6-a1c0-eaf0a5720f1e']]) {
                        sh "aws s3 cp $WORKSPACE/RPMS/noarch/${PACKAGE_NAME}-${VERSION}-${BUILD_NUMBER}.noarch.rpm s3://jenkins-bnr/package-repository/${PACKAGE_NAME}-${VERSION}-${BUILD_NUMBER}.noarch.rpm --region ap-southeast-1"
                    }
                }
            }
        }

        stage('Merge to Main') {
            steps {
                script {
                    sshagent(['sshgit']) {
                        echo "Start Merge to Main"

                        // Clone the repository to a temporary directory
                        sh "git clone ${params.REPO_URL} temp"

                        // Change to the cloned directory
                        dir('temp') {
                            // Initialize a new Git repository
                            sh 'git init'

                            // Set Git user.name and user.email configurations
                            sh "git config user.name 'Rainielle Maglaya'"
                            sh "git config user.email 'rainielle.maglaya@cambridge.org'"

                            // Fetch all remote branches
                            sh 'git fetch --all'

                            // Merge the specified branch into main with "theirs" strategy
                            def mergeResult = sh(script: "git merge -Xtheirs --no-ff origin/${params.BRANCH}", returnStatus: true)

                            // Check the merge result and take appropriate action
                            if (mergeResult == 0) {
                                echo "Merge successful."
                            } else {
                                echo "Merge conflict detected. Attempting to resolve..."
                                // Resolve merge conflicts here
                                sh 'git status'
                                sh 'git diff'
                                // Use 'git checkout' or other commands to resolve conflicts
                                // After resolving conflicts, commit the changes
                                sh 'git commit -m "Resolve merge conflicts"'
                            }

                            // Push the changes to the repository's main branch
                            sh 'git push origin main'
                        }

                        // Clean up the temporary directory
                        sh 'rm -rf temp'
                    }
                }
            }
        }
    }
}
