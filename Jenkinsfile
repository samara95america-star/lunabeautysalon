pipeline {
    agent any

    parameters {
        string(
            name: 'IMAGE_TAG',
            defaultValue: 'latest',
            description: 'ECR image tag to deploy'
        )
    }

    environment {
        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:${env.PATH}"
        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '584612873567'

        ECR_REPOSITORY = 'lunabeautysalon'

        ECS_CLUSTER    = 'lunabeautysalon'
        ECS_SERVICE    = 'lunabeautysalon-service'
        TASK_FAMILY    = 'lunabeautysalon'

        CONTAINER_NAME = 'lunabeautysalon'
    }

    stages {

        stage('Verify Environment') {
            steps {
                sh '''
                    echo "Checking AWS CLI..."
                    aws --version

                    echo "Checking jq..."
                    jq --version
                '''
            }
        }

        stage('Deployment Information') {
            steps {
                sh '''
                    echo "================================="
                    echo "Luna Beauty Salon CD"
                    echo "================================="
                    echo "Cluster: $ECS_CLUSTER"
                    echo "Service: $ECS_SERVICE"
                    echo "Task Family: $TASK_FAMILY"
                    echo "Image Tag: $IMAGE_TAG"
                    echo "================================="
                '''
            }
        }

        stage('Get Current Task Definition') {
            steps {
                sh '''
                    aws ecs describe-task-definition \
                      --task-definition $TASK_FAMILY \
                      --region $AWS_REGION \
                      --query taskDefinition \
                      > current-task-definition.json
                '''
            }
        }

        stage('Prepare New Task Definition') {
            steps {
                sh '''
                    IMAGE_URI="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:$IMAGE_TAG"

                    echo "Deploying image:"
                    echo "$IMAGE_URI"

                    jq \
                      --arg IMAGE "$IMAGE_URI" \
                      --arg CONTAINER "$CONTAINER_NAME" \
                      '
                      .containerDefinitions |= map(
                          if .name == $CONTAINER
                          then .image = $IMAGE
                          else .
                          end
                      )
                      |
                      del(
                          .taskDefinitionArn,
                          .revision,
                          .status,
                          .requiresAttributes,
                          .compatibilities,
                          .registeredAt,
                          .registeredBy
                      )
                      ' current-task-definition.json \
                      > new-task-definition.json

                    echo "New task definition prepared."
                '''
            }
        }

        stage('Register New Task Definition') {
            steps {
                script {

                    env.NEW_TASK_DEFINITION = sh(
                        script: '''
                            aws ecs register-task-definition \
                              --region $AWS_REGION \
                              --cli-input-json file://new-task-definition.json \
                              --query 'taskDefinition.taskDefinitionArn' \
                              --output text
                        ''',
                        returnStdout: true
                    ).trim()

                    echo "Created: ${env.NEW_TASK_DEFINITION}"
                }
            }
        }

        stage('Update ECS Service') {
            steps {
                sh '''
                    aws ecs update-service \
                      --cluster $ECS_CLUSTER \
                      --service $ECS_SERVICE \
                      --task-definition $NEW_TASK_DEFINITION \
                      --region $AWS_REGION
                '''
            }
        }

        stage('Wait for Deployment') {
            steps {
                sh '''
                    echo "Waiting for ECS deployment..."

                    aws ecs wait services-stable \
                      --cluster $ECS_CLUSTER \
                      --services $ECS_SERVICE \
                      --region $AWS_REGION

                    echo "ECS service is stable."
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    aws ecs describe-services \
                      --cluster $ECS_CLUSTER \
                      --services $ECS_SERVICE \
                      --region $AWS_REGION \
                      --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount,TaskDefinition:taskDefinition}' \
                      --output table
                '''
            }
        }
    }

    post {

        success {
            echo 'Deployment successful!'
        }

        failure {
            echo 'Deployment failed. Check Jenkins console output.'
        }
    }
}
