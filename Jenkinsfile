pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Enter git branch to deploy', trim: true)
        choice(name: 'ENV', choices: ['dev', 'stage', 'prod'], description: 'Choose deployment environment')
    }

    environment {
        FUNCTION_NAME = 'hey-world-demo'
        REGION = 'us-east-1'

        // Map environment to IAM execution role
        DEV_ROLE   = 'arn:aws:iam::529088259986:role/lambda_exec_role_dev'
        STAGE_ROLE = 'arn:aws:iam::529088259986:role/lambda_exec_role_stage'
        PROD_ROLE  = 'arn:aws:iam::529088259986:role/lambda_exec_role_prod'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${params.GIT_BRANCH}", url: 'https://github.com/rsync-cloud/aws-lambda.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('hello-world') {
                    sh 'pip install -r requirements.txt -t .'
                }
            }
        }

        stage('Zip Lambda Package') {
            steps {
                dir('hello-world') {
                    sh 'zip -r9 ../lambda_function.zip .'
                }
            }
        }

        stage('Deploy Lambda') {
            steps {
                script {
                    def roleArn = ""
                    if (params.ENV == "dev") {
                        roleArn = env.DEV_ROLE
                    } else if (params.ENV == "stage") {
                        roleArn = env.STAGE_ROLE
                    } else if (params.ENV == "prod") {
                        roleArn = env.PROD_ROLE
                    }

                    echo "Deploying Lambda to ${params.ENV} using role: ${roleArn}"

                    sh """
                        aws lambda create-function \
                          --function-name ${FUNCTION_NAME}-${params.ENV} \
                          --runtime python3.9 \
                          --role ${roleArn} \
                          --handler lambda_function.lambda_handler \
                          --zip-file fileb://lambda_function.zip \
                          --region ${REGION} \
                        || aws lambda update-function-code \
                          --function-name ${FUNCTION_NAME}-${params.ENV} \
                          --zip-file fileb://lambda_function.zip \
                          --region ${REGION}
                    """
                }
            }
        }

        stage('Setup CloudWatch Schedule') {
            steps {
                script {
                    def ruleName = "${FUNCTION_NAME}-${params.ENV}-schedule"
                    sh """
                        aws events put-rule \
                          --name ${ruleName} \
                          --schedule-expression "rate(5 minutes)" \
                          --state ENABLED \
                          --region ${REGION}
                    """
                }
            }
        }
    }
}
