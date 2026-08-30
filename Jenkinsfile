pipeline {
    agent any
    environment {
        AWS_REGION      = 'eu-north-1'
        ECR_REGISTRY    = '305018987435.dkr.ecr.eu-north-1.amazonaws.com'
        ECR_REPO        = 'cicd-project'
        ECS_CLUSTER     = 'cicd-app-cluster'
        ECS_SERVICE     = 'cicd-app-taskdefinition-service-rch9ch1c'
        TASK_FAMILY     = 'cicd-app-taskdefinition'
        CONTAINER_NAME  = 'cicd-container'
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/ahmmedtarek/jenkins-cicd-project'
            }
        }
        stage('Build') {
            steps {
                sh 'docker build -t cicd-app:${BUILD_NUMBER} .'
            }
        }
        stage('Test') {
            steps {
                sh '''
                    docker rm -f test-container || true
                    docker run -d --name test-container -p 8001:80 cicd-app:${BUILD_NUMBER}
                    sleep 2
                    curl -f http://localhost:8001
                    docker rm -f test-container
                '''
            }
        }
        stage('Push to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    docker tag cicd-app:${BUILD_NUMBER} ${ECR_REGISTRY}/${ECR_REPO}:${BUILD_NUMBER}
                    docker push ${ECR_REGISTRY}/${ECR_REPO}:${BUILD_NUMBER}
                '''
            }
        }
        stage('Deploy to ECS') {
            steps {
                sh '''
                    # 1. Fetch the current task definition
                    aws ecs describe-task-definition \
                        --task-definition ${TASK_FAMILY} \
                        --region ${AWS_REGION} \
                        --query 'taskDefinition' > current-task-def.json

                    # 2. Update the image for our container, strip fields register-task-definition rejects
                    jq --arg IMAGE "${ECR_REGISTRY}/${ECR_REPO}:${BUILD_NUMBER}" \
                        --arg NAME "${CONTAINER_NAME}" \
                        '.containerDefinitions = (.containerDefinitions | map(if .name == $NAME then .image = $IMAGE else . end))
                         | del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)' \
                        current-task-def.json > new-task-def.json

                    # 3. Register the new task definition revision
                    NEW_TASK_DEF_ARN=$(aws ecs register-task-definition \
                        --region ${AWS_REGION} \
                        --cli-input-json file://new-task-def.json \
                        --query 'taskDefinition.taskDefinitionArn' \
                        --output text)

                    echo "Registered new task definition: ${NEW_TASK_DEF_ARN}"

                    # 4. Update the service to use the new revision
                    aws ecs update-service \
                        --cluster ${ECS_CLUSTER} \
                        --service ${ECS_SERVICE} \
                        --task-definition ${NEW_TASK_DEF_ARN} \
                        --health-check-grace-period-seconds 180 \
                        --force-new-deployment \
                        --region ${AWS_REGION}

                    # 5. Wait for the service to stabilize (new task healthy, old task drained)
                    aws ecs wait services-stable \
                        --cluster ${ECS_CLUSTER} \
                        --services ${ECS_SERVICE} \
                        --region ${AWS_REGION}

                    echo "Deployment complete and stable."
                '''
            }
        }
    }
}
