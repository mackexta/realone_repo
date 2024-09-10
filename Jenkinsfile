pipeline {
    agent any
    environment {
        VERSION = "${env.BUILD_ID}"
        AWS_ACCOUNT_ID = "730335412936"
        AWS_DEFAULT_REGION = "us-east-1"
        IMAGE_REPO_NAME = "nora_pipeline"
        IMAGE_TAG = "${env.BUILD_ID}"
        REPOSITORY_URI = "730335412936.dkr.ecr.us-east-1.amazonaws.com/nora_pipeline"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/Orangeorobosa123/realone-repo.git', branch: 'master'  
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'cd SampleWebApp && mvn clean install'
            }
        }

        stage('Test') {
            steps {
                sh 'cd SampleWebApp && mvn test'
            }
        }

        stage('Code Quality Scan') {
            steps {
                withSonarQubeEnv('sonar-scanner') {
                    sh "mvn -f SampleWebApp/pom.xml sonar:sonar"
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Logging into AWS ECR') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('aws_secret_access_key')
            }
            steps {
                script {
                    sh """
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | \
                        docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
                    """
                }
            }
        }

        stage('Building Image') {
            steps {
                script {
                    dockerImage = docker.build("${IMAGE_REPO_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Pushing to ECR') {
            steps {
                script {
                    sh """
                        docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:${IMAGE_TAG}
                        docker push ${REPOSITORY_URI}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Pull Image & Deploying UI Application on EKS Cluster DEV') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('aws_secret_access_key')
            }
            steps {
                script {
                    dir('kubernetes/') {
                        sh 'aws eks update-kubeconfig --name myAppp-eks-cluster --region us-east-1'
                        sh """
                            aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | \
                            docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
                        """
                        sh 'helm upgrade --install --set image.repository="${REPOSITORY_URI}" --set image.tag="${IMAGE_TAG}" myjavaapp myapp/'
                    }
                }
            }
        }
    }
}
