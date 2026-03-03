pipeline {
    agent any

    environment {
        AWS_REGION = "us-east-1"
        ECR_REGISTRY = "237598454971.dkr.ecr.us-east-1.amazonaws.com"
        KUBE_NAMESPACE = "dev"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/anasadam20/eks-project.git'
            }
        }

        stage('Build & Push Images') {
            steps {
                script {
                    def services = ["users", "items", "orders"]
                    for (svc in services) {
                        sh """
                        cd ${svc}
                        docker build -t $ECR_REGISTRY/${svc}:$BUILD_NUMBER .
                        aws ecr get-login-password --region $AWS_REGION | \
                          docker login --username AWS --password-stdin $ECR_REGISTRY
                        docker push $ECR_REGISTRY/${svc}:$BUILD_NUMBER
                        cd ..
                        """
                    }
                }
            }
        }

        stage('Helm Deploy') {
            steps {
                script {
                    def services = ["users", "items", "orders"]
                    for (svc in services) {
                        sh """
                        helm upgrade --install ${svc} ./helm/${svc} \
                          --namespace $KUBE_NAMESPACE \
                          --set image.repository=$ECR_REGISTRY/${svc} \
                          --set image.tag=$BUILD_NUMBER
                        """
                    }
                }
            }
        }

        stage('Smoke Test') {
            steps {
                script {
                    def alb_dns = sh(script: "kubectl get ingress namespace-ingress -n $KUBE_NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                    def paths = ["users", "items", "orders"]
                    for (p in paths) {
                        sh "curl -s http://$alb_dns/$p | grep '<html>'"
                    }
                }
            }
        }
    }
}
