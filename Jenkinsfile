pipeline {
    agent {
        label 'terraform'
    }

    parameters{
        text(name: 'CAT_FE_VERSION', defaultValue: 'latest', description: 'Version of CAT FE.')
        text(name: 'CAT_BE_VERSION', defaultValue: 'latest', description: 'Version of CAT BE.')
        text(name: 'ASG_NAME', defaultValue: 'app-stack-asg', description: 'Name of the current ASG.')
        text(name: 'IMAGE_REGISTRY', defaultValue: '339712946908.dkr.ecr.ap-southeast-1.amazonaws.com', description: 'ECR registry store artifact')
    }

    stages {
        stage('Preconfig') {
            steps {
                sh 'sudo systemctl start docker'
                sh 'sudo docker login ${IMAGE_REGISTRY} --username AWS --password $(aws ecr get-login-password --region ap-southeast-1)'
            }
        }

        stage('Change image tag') {
            steps {
                sh 'git checkout main'
                sh '''
                    # Build image name
                    export CAT_FE_IMAGE=${IMAGE_REGISTRY}/cat-fe:${CAT_FE_VERSION}
                    export CAT_BE_IMAGE=${IMAGE_REGISTRY}/cat-be:${CAT_BE_VERSION}

                    # Replace value
                    yq e -i '.services.fe.image=env(CAT_FE_IMAGE)' ./cat/docker-compose.yml
                    yq e -i '.services.be.image=env(CAT_BE_IMAGE)' ./cat/docker-compose.yml
                    yq e -i '.services.migration.image=env(CAT_BE_IMAGE)' ./cat/docker-compose.yml
                '''
            }
        }

        stage('Run migration') {
            steps {
                dir('./cat') {
                    sh '''
                        source /opt/inject-ssm-secrets/bin/activate
                        export AWS_DEFAULT_REGION=ap-southeast-1
                        python /opt/inject-ssm-secrets/inject-ssm-secrets.py --path /cat-be --output-type file --output-file cat-be.env
                    '''
                    sh 'sudo docker compose --profile migration up'
                }
            }
        }

        stage('Apply image tag') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'git-cred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                    sh '''
                        git config user.name "Release Bot"
                        git config user.email "jenkins@local.mail"
                        git add ./cat/docker-compose.yml
                        git commit -m "Bump cat-fe to $CAT_FE_VERSION and cat-be to $CAT_BE_VERSION"
                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/vucong2409/sc-lab-3-app-stack main
                    '''
                }
            }
        }


        stage('Run instance refresh') {
            steps {
                sh 'aws autoscaling start-instance-refresh --auto-scaling-group-name ${ASG_NAME}'
            }
        }
    }

    post {
        always {
            cleanWs(cleanWhenNotBuilt: false,
            deleteDirs: true,
            disableDeferredWipeout: true,
            notFailBuild: true)
        }
    }
}
