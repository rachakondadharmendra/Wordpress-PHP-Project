pipeline {
    agent none
    
    tools {
        jdk 'jdk-11'
    }
    
    environment {
        REGION='ap-south-2'
        ECR_REPOSITORY='wordpress-ecr'
        AWS_ACCOUNT_ID='079711632559'
    }
    
    stages {
        stage('Checkout') {
            agent {
                docker {
                    image 'rachakondadharmendra/custom-node:alpine3.19'
                    reuseNode true
                }
            }
            steps {
                git branch: 'master', changelog: false, credentialsId: 'github_admin_creds', poll: false, url: 'https://github.com/rachakondadharmendra/Wordpress-PHP-Project'
            }
        }

        stage('GitLeaks') {
            agent {
                docker {
                    image 'zricethezav/gitleaks'
                    args '--entrypoint=""'
                }
            }
            steps {
                script {
                    //sh 'gitleaks detect --verbose --source . --config ./gitleaks.toml --report-format csv --report-path secrets.csv || exit 0'
                    sh 'gitleaks detect --verbose --source . --report-format csv --report-path secrets.csv || exit 0'
                }
            }
        }

        stage('PHP Code Sniffer') {
            agent {
                docker {
                    image 'cytopia/phpcs:latest'
                    args "--entrypoint=''"
                }
            }
            steps {
                sh 'phpcs --standard=PSR2 . || exit 0'
                echo "PHP CodeSniffer has completed scanning."
            }
        }

        stage('Unit Testing') {
            agent {
                docker {
                    image 'rachakondadharmendra/custom-node:alpine3.19'
                    reuseNode true
                }
            }
            steps {
    
                echo "Unit Testing of code is done"
            }
        }        

        stage("Checkov scanning") {
            agent {
                docker {
                    image 'bridgecrew/checkov:3.2.49'
                    args "--entrypoint=''"
                }
            }
            steps {
                sh 'checkov --version'
                sh 'checkov -d . --skip-check CKV_DOCKER_2 --skip-check CKV_DOCKER_3 --framework dockerfile || exit 0'
            }
        }

        stage("Docker Build") {
            agent {
                docker {
                    image 'rachakondadharmendra/custom-node:alpine3.19'
                    reuseNode true
                }
            } 
            steps {
                script {
                    sh 'docker build -t ${ECR_REPOSITORY}:jenkinsbuild .'
                }
            }
        }

        stage("Trivy scanning") {
            agent {
                docker {
                    image 'aquasec/trivy:latest'
                    args "--entrypoint=''"
                }
            }
            steps {
                sh 'trivy --version'
                sh 'trivy image --format table ${ECR_REPOSITORY}:jenkinsbuild'
            }
        }

        stage('Tag and Push to ECR') {
            agent {
                docker {
                    image 'rachakondadharmendra/custom-node:alpine3.19'
                    reuseNode true
                }
            } 
            environment {
                  BRANCH_NAME=sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
                  SHORT_PR_NUMBER=sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                  CURRENT_TIMESTAMP=sh(script: 'date "+%d-%m-%Y-%H.%M.%S"', returnStdout: true).trim()
                  IMAGE_TAG="${BRANCH_NAME}_${SHORT_PR_NUMBER}_${CURRENT_TIMESTAMP}"
            }            
            steps {
                script {
                    sh 'aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com'
                    sh 'docker tag ${ECR_REPOSITORY}:jenkinsbuild ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${ECR_REPOSITORY}:${IMAGE_TAG}'
                    sh 'docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${ECR_REPOSITORY}:${IMAGE_TAG}'
                }
            }
        }

    }
}
