pipeline {
    agent any
    environment {
        npm_config_cache = "npm-cache"
    }
    stages {
        stage("Check versions") {
            steps {
                sh '''
                    aws --version
                    eksctl version
                    kubectl version 2>&1 | tr -d '\n'
                    docker --version
                    node --version
                    npm --version
                    hadolint --version
                    ls
                '''
            }
        }
        stage("Build") {
            steps {
                sh "npm install"
            }
        }
        stage("Test") {
            steps {
                sh "CI=true npm test"
            }
        }
        stage("Release") {
            steps {
                sh "npm run build"
            }
        }
        stage("Lint") {
            steps {
                sh "hadolint infra/Dockerfile"
            }
        }
        stage("Dockerize app") {
            steps {
                sh "docker build -t clequinio/aws-k8s-react-app:${env.BUILD_TAG} ."
            }
        }
        stage("docker push") {
            steps {
                script {
                    sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 860597918607.dkr.ecr.us-east-1.amazonaws.com"
                    sh "docker tag clequinio/aws-k8s-react-app:${env.BUILD_TAG} 860597918607.dkr.ecr.us-east-1.amazonaws.com/react_eks:latest"
                    sh "docker push 860597918607.dkr.ecr.us-east-1.amazonaws.com/react_eks:latest"
                }
            }
        }
        stage("Map kubectl to the k8s aws cluster and configure") {
            steps {
                withAWS(credentials: "aws_creds", region: "us-east-1") {
                    sh "aws eks --region us-east-1 update-kubeconfig --name react_eks_cluster"
                    sh "kubectl config use-context arn:aws:eks:us-east-1:860597918607:cluster/react_eks_cluster"
                    sh "kubectl apply -f infra/k8s-config.yml"
                }
            }
        }
        stage("Deploy the new app dockerized") {
            steps {
                withAWS(credentials: "aws_creds", region: "us-east-1") {
                    sh "kubectl set image deployment/aws-k8s-react-app-deployment aws-k8s-react-app=clequinio/aws-k8s-react-app:${env.BUILD_TAG}"
                }
            }
        }
        stage("Test deployment") {
            steps {
                withAWS(credentials: "aws_creds", region: "us-east-1") {
                    sh "kubectl get nodes"
                    sh "kubectl get deployment"
                    sh "kubectl get pod -o wide"
                    sh "kubectl get service/service-aws-k8s-react-app"
                    sh "curl \$(kubectl get service/service-aws-k8s-react-app --output jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
                }
            }
        }
    }
}
