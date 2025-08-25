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
                    def execRole = ''

                    if (params.ENV == 'dev') {
                        awsCredId = 'aws-cred-dev'
                        execRole = 'lambda_exec_role_dev'
                    } else if (params.ENV == 'stage') {
                        awsCredId = 'aws-cred-stage'
                        execRole = 'lambda_exec_role_stage'
                    } else if (params.ENV == 'prod') {
                        awsCredId = 'aws-cred-prod'
                        execRole = 'lambda_exec_role_prod'
                    }

                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: awsCredId]]) {
                        sh """
                          export AWS_DEFAULT_REGION=${REGION}
                          cd aws-lambda

                          # Update code
                          aws lambda update-function-code \
                            --function-name ${FUNCTION_NAME} \
                            --zip-file fileb://lambda_package.zip

                          # Ensure execution role is correct per environment
                          aws lambda update-function-configuration \
                            --function-name ${FUNCTION_NAME} \
                            --role arn:aws:iam::\$(aws sts get-caller-identity --query Account --output text):role/${execRole}
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
