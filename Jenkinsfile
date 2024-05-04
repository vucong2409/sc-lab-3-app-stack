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
            sh 'sudo systemctl start docker'
            sh 'sudo docker login ${IMAGE_REGISTRY} --username AWS --password $(aws ecr get-login-password --region ap-southeast-1)'
        }
        stage('Run migration') {
            dir('./app-stack/cat') {
                sh 'source /opt/inject-ssm-secrets/bin/activate'
                sh 'python /opt/inject-ssm-secrets/inject-ssm-secrets.py --path /cat-be --output-type file --output-file cat-be.env'
                sh 'sudo docker compose --profile migration up'
            }
        }
        stage('Change image tag') {

        }
        stage('Run instance refresh') {
            sh 'aws autoscaling start-instance-refresh --auto-scaling-group-name ${ASG_NAME}'
        }
    }
}
