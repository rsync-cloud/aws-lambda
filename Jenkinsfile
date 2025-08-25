pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Enter git branch to deploy', trim: true)
        choice(name: 'ENV', choices: ['dev', 'stage', 'prod'], description: 'Choose deployment environment')
    }

    environment {
        FUNCTION_NAME = 'hey-world-demo'
        REGION = 'us-east-1'
        SONAR_PROJECT_KEY = "aws-lambda"
        SONAR_ORG = "your-org"
    }

    stages {

        stage('Checkout') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                      rm -rf aws-lambda
                      BRANCH_NAME=${GIT_BRANCH#origin/}
                      echo "Cloning branch: $BRANCH_NAME"
                      git clone -b $BRANCH_NAME https://${GIT_USER}:${GIT_TOKEN}@github.com/rsync-cloud/aws-lambda.git
                      cd aws-lambda
                      ls -la
                    '''
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                  cd aws-lambda/hello-world
                  pip install -r requirements.txt -t .
                '''
            }
        }

        stage('Package Lambda') {
            steps {
                sh '''
                  cd aws-lambda/hello-world
                  zip -r9 ../lambda_package.zip .
                '''
            }
        }

        // Commented Sonar Stage
        /*
        stage('SonarQube Analysis') {
            environment {
                SONAR_TOKEN = credentials('SONAR_TOKEN')
            }
            steps {
                sh '''
                  sonar-scanner \
                    -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                    -Dsonar.organization=${SONAR_ORG} \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=https://sonarcloud.io \
                    -Dsonar.login=${SONAR_TOKEN}
                '''
            }
        }
        */

        stage('Deploy to AWS Lambda') {
            steps {
                script {
                    def awsCredId = ''
                    def lambdaRole = ''
                    
                    if (params.ENV == 'dev') {
                        awsCredId = 'aws-cred-dev'
                        lambdaRole = 'arn:aws:iam::529088259986:role/lambda_exec_role_dev'
                    } else if (params.ENV == 'stage') {
                        awsCredId = 'aws-cred-stage'
                        lambdaRole = 'arn:aws:iam::529088259986:role/lambda_exec_role_stage'
                    } else if (params.ENV == 'prod') {
                        awsCredId = 'aws-cred-prod'
                        lambdaRole = 'arn:aws:iam::529088259986:role/lambda_exec_role_prod'
                    }

                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: awsCredId]]) {
                        sh """
                          export AWS_DEFAULT_REGION=${REGION}
                          cd aws-lambda

                          # Create Lambda function if it doesn't exist
                          aws lambda get-function --function-name ${FUNCTION_NAME} || \
                          aws lambda create-function \
                            --function-name ${FUNCTION_NAME} \
                            --runtime python3.13 \
                            --role ${lambdaRole} \
                            --handler hello-world.lambda_function.lambda_handler \
                            --zip-file fileb://lambda_package.zip

                          # Update Lambda code
                          aws lambda update-function-code \
                            --function-name ${FUNCTION_NAME} \
                            --zip-file fileb://lambda_package.zip
                        """
                    }
                }
            }
        }

        stage('Setup CloudWatch Schedule') {
            steps {
                script {
                    def awsCredId = ''
                    if (params.ENV == 'dev') {
                        awsCredId = 'aws-cred-dev'
                    } else if (params.ENV == 'stage') {
                        awsCredId = 'aws-cred-stage'
                    } else if (params.ENV == 'prod') {
                        awsCredId = 'aws-cred-prod'
                    }

                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: awsCredId]]) {
                        sh '''
                          export AWS_DEFAULT_REGION=${REGION}
                          RULE_NAME="hello-world-schedule"
                          SCHEDULE="rate(5 minutes)"

                          # Create or update CloudWatch rule
                          aws events put-rule \
                            --name $RULE_NAME \
                            --schedule-expression "$SCHEDULE" \
                            --state ENABLED

                          # Add Lambda as target of the rule
                          aws events put-targets \
                            --rule $RULE_NAME \
                            --targets "Id"="1","Arn"=$(aws lambda get-function --function-name ${FUNCTION_NAME} --query 'Configuration.FunctionArn' --output text)

                          # Grant permission for Events to invoke Lambda
                          aws lambda add-permission \
                            --function-name ${FUNCTION_NAME} \
                            --statement-id "cw-invoke" \
                            --action 'lambda:InvokeFunction' \
                            --principal events.amazonaws.com \
                            --source-arn $(aws events describe-rule --name $RULE_NAME --query 'Arn' --output text) || true
                        '''
                    }
                }
            }
        }
    }
}
