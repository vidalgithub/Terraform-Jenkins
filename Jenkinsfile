pipeline {
    parameters {
        choice(name: 'ACTION', choices: ['APPLY', 'DESTROY'], description: 'Choose action to perform')
        booleanParam(name: 'autoApprove', defaultValue: false, description: 'Automatically run the selected action after generating plan?')
    }
    environment {
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
    }

    agent {
        docker { 
            image 'hashicorp/terraform:1.9.3' 
            args '-e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY --entrypoint=""'  // Pass AWS credentials and reset entrypoint
        }
    }

    stages {
        stage('Verify Docker Setup') {
            steps {
                sh 'terraform --version'  // Ensure Terraform command works
            }
        }

        stage('Checkout') {
            steps {
                script {
                    dir("terraform") {
                        git "https://github.com/vidalgithub/Terraform-Jenkins.git"
                    }
                }
            }
        }

        stage('Plan') {
            when {
                expression { return params.ACTION == 'APPLY' }
            }
            steps {
                sh 'cd terraform && terraform init'
                sh 'cd terraform && terraform plan -out=tfplan'
                sh 'cd terraform && terraform show -no-color tfplan > tfplan.txt'
            }
        }

        stage('Plan Destroy') {
            when {
                expression { return params.ACTION == 'DESTROY' }
            }
            steps {
                sh 'cd terraform && terraform init'
                sh 'cd terraform && terraform plan -destroy -out=tfplan-destroy'
                sh 'cd terraform && terraform show -no-color tfplan-destroy > tfplan-destroy.txt'
            }
        }

        stage('Approval') {
            when {
                not {
                    equals expected: true, actual: params.autoApprove
                }
            }
            steps {
                script {
                    def planFile = params.ACTION == 'APPLY' ? 'terraform/tfplan.txt' : 'terraform/tfplan-destroy.txt'
                    def plan = readFile planFile
                    input message: "Do you want to ${params.ACTION.toLowerCase()} the resources?",
                          parameters: [text(name: 'Plan', description: 'Please review the plan', defaultValue: plan)]
                }
            }
        }

        stage('Apply') {
            when {
                expression { return params.ACTION == 'APPLY' }
            }
            steps {
                sh 'cd terraform && terraform apply -input=false tfplan'
            }
        }

        stage('Destroy') {
            when {
                expression { return params.ACTION == 'DESTROY' }
            }
            steps {
                sh 'cd terraform && terraform apply -destroy -input=false tfplan-destroy'
            }
        }

        stage('Verify') {
            when {
                expression { return params.ACTION == 'APPLY' }
            }
            agent {
                docker { 
                    image 'amazon/aws-cli' 
                }
            }
            steps {
                script {
                    sh '''
                        aws ec2 describe-instances --filters "Name=tag:Name,Values=TF-Jenkins-Instance" --query "Reservations[*].Instances[*].{Instance:InstanceId,Name:Tags[?Key=='Name'].Value|[0]}" --output json
                    '''
                }
            }
        }
    }
}
