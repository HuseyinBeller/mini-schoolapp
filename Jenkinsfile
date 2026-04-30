pipeline {
    agent { label 'docker-node' } // Node with Docker and AWS CLI
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['DEV', 'STAGING', 'PROD'], description: 'Target environment')
        string(name: 'AWS_ACCOUNT_ID', defaultValue: '211125625059', description: 'Target AWS Account ID')
        string(name: 'ROLE_NAME', defaultValue: 'Classof25-STS-Role', description: 'IAM Role to assume in target account')
        string(name: 'AWS_REGION', defaultValue: 'eu-central-1', description: 'AWS Region')
        string(name: 'ECR_REPO_NAME', defaultValue: 'classof25repo', description: 'Base ECR repository name')
        string(name: 'EC2_INSTANCE_ID', defaultValue: 'i-0bd3f61fdd14344a8', description: 'EC2 Instance ID')
        string(name: 'EC2_SSH_USER', defaultValue: 'ubuntu', description: 'SSH user for EC2 instance')
        string(name: 'HOST_PORT', defaultValue: '80', description: 'Host port for the Docker container')
    }
    stages {
        stage('Validate Input') {
            steps {
                script {
                    if (!params.ENVIRONMENT || !['DEV', 'STAGING', 'PROD'].contains(params.ENVIRONMENT)) {
                        error "Invalid ENVIRONMENT. Must be DEV, STAGING, or PROD."
                    }
                    if (!params.AWS_ACCOUNT_ID || params.AWS_ACCOUNT_ID ==~ /[^0-9]/ || params.AWS_ACCOUNT_ID.length() != 12) {
                        error "Invalid AWS_ACCOUNT_ID. Must be a 12-digit number."
                    }
                    if (!params.ROLE_NAME) {
                        error "ROLE_NAME is required."
                    }
                    if (!params.AWS_REGION) {
                        error "AWS_REGION is required."
                    }
                    if (!params.ECR_REPO_NAME) {
                        error "ECR_REPO_NAME is required."
                    }
                    if (!params.EC2_INSTANCE_ID) {
                        error "EC2_INSTANCE_ID is required."
                    }
                    if (!params.EC2_SSH_USER) {
                        error "EC2_SSH_USER is required."
                    }
                    if (!params.HOST_PORT || params.HOST_PORT ==~ /[^0-9]/ || params.HOST_PORT.toInteger() < 1 || params.HOST_PORT.toInteger() > 65535) {
                        error "Invalid HOST_PORT. Must be a number between 1 and 65535."
                    }
                }
            }
        }
        // stage('stsInitial') {
        //     steps {
        //         script {
        //             // Initialize AWS CLI with default region
        //             sh "aws sts get-caller-identity --output json"
        //         }
        //     }
        // }
        stage('Assume Role') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'classof25-sts-user',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        def roleArn = "arn:aws:iam::${params.AWS_ACCOUNT_ID}:role/${params.ROLE_NAME}"
                        def sessionName = "JenkinsSession-${env.BUILD_ID}"
                        def stsOutput = sh(script: """
                            aws sts assume-role \
                                --role-arn ${roleArn} \
                                --role-session-name ${sessionName} \
                                --output json
                        """, returnStdout: true).trim()
                        def stsJson = readJSON text: stsOutput
                        env.AWS_ACCESS_KEY_ID = stsJson.Credentials.AccessKeyId
                        env.AWS_SECRET_ACCESS_KEY = stsJson.Credentials.SecretAccessKey
                        env.AWS_SESSION_TOKEN = stsJson.Credentials.SessionToken
                    }
                }
            }
        }
        stage('stsInitial02') {
            steps {
                script {
                    // Initialize AWS CLI with default region 
                    sh "aws sts get-caller-identity --output json"
                }
            }
        }
        stage('Validate/Create ECR Repository') {
            steps {
                script {
                    def repoName = "${params.ECR_REPO_NAME}-${params.ENVIRONMENT.toLowerCase()}"
                    env.ECR_REPO_URL = "${params.AWS_ACCOUNT_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com/${repoName}"
                    /// Check if repository exists
                    def repoOutput = sh(script: """
                        aws ecr describe-repositories \
                            --region ${params.AWS_REGION} \
                            --repository-names ${repoName} \
                            --output json 2>/dev/null || echo '{}'
                    """, returnStdout: true).trim()
                    def repoJson = readJSON text: repoOutput
                    if (!repoJson.repositories || repoJson.repositories.size() == 0) {
                        echo "ECR repository ${repoName} does not exist. Creating..."
                        def createStatus = sh(script: """
                            aws ecr create-repository \
                                --region ${params.AWS_REGION} \
                                --repository-name ${repoName} \
                                --output json
                        """, returnStatus: true)
                        if (createStatus != 0) {
                            error "Failed to create ECR repository ${repoName}. Check permissions or if it already exists."
                        } 
                        echo "ECR repository ${repoName} created successfully."
                    } else {
                        echo "ECR repository ${repoName} already exists."
                    }
                }
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                script {
                    def imageTag = params.ENVIRONMENT.toLowerCase()
                    def fullImage = "${env.ECR_REPO_URL}:${imageTag}-${env.BUILD_ID}"
                    // Build Docker image
                    sh "docker build -t classof25:${imageTag}-${env.BUILD_ID} ."
                    // Authenticate to ECR
                    def loginStatus = sh(script: """
                        aws ecr get-login-password \
                            --region ${params.AWS_REGION} | \
                            docker login \
                                --username AWS \
                                --password-stdin ${params.AWS_ACCOUNT_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com
                    """, returnStatus: true)
                    if (loginStatus != 0) {
                        error "Failed to authenticate to ECR. Check permissions for ecr:GetAuthorizationToken."
                    }
                    // Tag and push image
                    sh """
                        docker tag classof25:${imageTag}-${env.BUILD_ID} ${fullImage}
                        docker push ${fullImage}
                    """
                }
            }
        }
        // stage('Validate EC2 Instance') {
        //     steps {
        //         script {
        //             // Check instance accessibility
        //             def instanceStatus = sh(script: """
        //                 aws ec2 describe-instances \
        //                     --region ${params.AWS_REGION} \
        //                     --instance-ids ${params.EC2_INSTANCE_ID} \
        //                     --query 'Reservations[0].Instances[0].State.Name' \
        //                     --output text
        //             """, returnStdout: true, returnStatus: true)
        //             if (instanceStatus != 0) {
        //                 error "EC2 instance ${params.EC2_INSTANCE_ID} does not exist or is not accessible."
        //             }
        //             // Get instance details
        //             def instanceOutput = sh(script: """
        //                 aws ec2 describe-instances \
        //                     --region ${params.AWS_REGION} \
        //                     --instance-ids ${params.EC2_INSTANCE_ID} \
        //                     --output json
        //             """, returnStdout: true).trim()
        //             def instanceJson = readJSON text: instanceOutput
        //             if (!instanceJson.Reservations || instanceJson.Reservations.size() == 0 || !instanceJson.Reservations[0].Instances || instanceJson.Reservations[0].Instances.size() == 0) {
        //                 error "EC2 instance ${params.EC2_INSTANCE_ID} not found."
        //             }
        //             def state = instanceJson.Reservations[0].Instances[0].State.Name
        //             if (state != 'running') {
        //                 error "EC2 instance ${params.EC2_INSTANCE_ID} is not in 'running' state. Current state: ${state}"
        //             }
        //             env.EC2_IP = instanceJson.Reservations[0].Instances[0].PublicIpAddress ?: instanceJson.Reservations[0].Instances[0].PrivateIpAddress
        //             if (!env.EC2_IP) {
        //                 error "Could not retrieve IP address for EC2 instance ${params.EC2_INSTANCE_ID}."
        //             }
        //             echo "EC2 instance ${params.EC2_INSTANCE_ID} is running with IP address ${env.EC2_IP}."
        //         }
        //     }
        // }
    
        // stage('Deploy to EC2') {
        //     steps {
        //         withCredentials([sshUserPrivateKey(
        //             credentialsId: 'docker-server-key',
        //             keyFileVariable: 'SSH_KEY',
        //             usernameVariable: 'SSH_USERNAME'
        //         )]) {
        //             script {
        //                 def fullImage = "${env.ECR_REPO_URL}:${params.ENVIRONMENT.toLowerCase()}-${env.BUILD_ID}"
        //                 def containerName = "classof25-${params.ENVIRONMENT.toLowerCase()}"
        //                 // Create .ssh directory in workspace!
        //                 sh "mkdir -p ${env.WORKSPACE}/.ssh"
        //                 // Generate known_hosts file non-interactively
        //                 def keyscanStatus = sh(script: """
        //                     timeout 10s ssh-keyscan -t rsa,ecdsa,ed25519 -H ${env.EC2_IP} >> ${env.WORKSPACE}/.ssh/known_hosts
        //                 """, returnStatus: true)
        //                 if (keyscanStatus != 0) {
        //                     error "Failed to fetch EC2 host key for ${env.EC2_IP}. Ensure the instance is reachable and SSH is enabled."
        //                 }
        //                 // Write SSH commands to a script
        //                 writeFile file: 'deploy.sh', text: """
        //                     #!/bin/bash
        //                     set -e
        //                     # Authenticate to ECR (instance profile handles permissions)
        //                     aws ecr get-login-password --region ${params.AWS_REGION} | \
        //                         docker login --username AWS --password-stdin ${params.AWS_ACCOUNT_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com
        //                     # Pull the new image
        //                     docker pull ${fullImage}
        //                     # Clean up previous images (keep the newly pulled image)
        //                     docker images ${env.ECR_REPO_URL} --format '{{.Tag}}' | grep -v "${params.ENVIRONMENT.toLowerCase()}-${env.BUILD_ID}" | xargs -I {} docker rmi ${env.ECR_REPO_URL}:{} || true
        //                     # Stop and remove existing container (if any)
        //                     docker stop ${containerName} || true
        //                     docker rm ${containerName} || true
        //                     # Run the new container
        //                     docker run -d --name ${containerName} -p ${params.HOST_PORT}:80 ${fullImage}
        //                     # Prune unused containers
        //                     docker system prune -f || true
        //                 """
        //                 // Copy and execute script on EC2.
        //                 sh """
        //                     chmod 600 \$SSH_KEY
        //                     scp -i \$SSH_KEY -o UserKnownHostsFile=${env.WORKSPACE}/.ssh/known_hosts deploy.sh ${params.EC2_SSH_USER}@${env.EC2_IP}:~/deploy.sh
        //                     ssh -i \$SSH_KEY -o UserKnownHostsFile=${env.WORKSPACE}/.ssh/known_hosts ${params.EC2_SSH_USER}@${env.EC2_IP} 'chmod +x ~/deploy.sh && ~/deploy.sh'
        //                     rm -f \$SSH_KEY
        //                     rm -rf ${env.WORKSPACE}/.ssh
        //                 """
        //             }
        //         }
        //     }
        // }
    }
    post {
        failure {
            echo "Pipeline failed. Please check the logs for details."
        }
        always {
            // Clean up temporary credentials
            script {
                env.AWS_ACCESS_KEY_ID = ''
                env.AWS_SECRET_ACCESS_KEY = ''
                env.AWS_SESSION_TOKEN = ''
            }
        }
    }
}