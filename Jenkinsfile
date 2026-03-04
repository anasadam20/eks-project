pipeline {
    agent any

    environment {
        AWS_REGION = "us-east-1"
        ECR_REGISTRY = "237598454971.dkr.ecr.us-east-1.amazonaws.com"
        KUBE_NAMESPACE = "dev"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/anasadam20/eks-project.git'
            }
        }

        stage('Build & Push Images') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'fbc5969b-681f-4e13-bcc6-fdf5e1446cbf']]) {
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
        }

        stage('Helm Deploy') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'fbc5969b-681f-4e13-bcc6-fdf5e1446cbf']]) {
                    script {
                        sh "aws sts get-caller-identity"
                        sh "aws eks update-kubeconfig --region $AWS_REGION --name my-eks-cluster"
                        sh "kubectl auth can-i list secrets -n dev"

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
        }

        stage('Verify Rollout') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'fbc5969b-681f-4e13-bcc6-fdf5e1446cbf']]) {
                    script {
                        def services = ["users", "items", "orders"]
                        for (svc in services) {
                            sh "kubectl rollout status deployment/${svc} -n $KUBE_NAMESPACE --timeout=60s"
                        }
                    }
                } // ✅ properly closed withCredentials
            }
        }

        stage('Smoke Test') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'fbc5969b-681f-4e13-bcc6-fdf5e1446cbf']]) {
                    script {
                        sh "aws eks update-kubeconfig --region $AWS_REGION --name my-eks-cluster"
                        sh "kubectl auth can-i get ingresses -n $KUBE_NAMESPACE"

                        def alb_dns = sh(script: "kubectl get ingress namespace-ingress -n $KUBE_NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                        def paths = ["users", "items", "orders"]

                        for (p in paths) {
                            def retries = 3
                            def success = false
                            for (int i = 0; i < retries; i++) {
                                def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://$alb_dns/$p", returnStdout: true).trim()
                                echo "Attempt ${i+1}: /$p → HTTP $status"
                                if (status == "200") {
                                    success = true
                                    sh "curl -s http://$alb_dns/$p | head -n 10"
                                    break
                                }
                                sleep 5
                            }
                            if (!success) {
                                error("Smoke test failed for /$p after ${retries} attempts")
                            }
                        }
                    }
                }
            }
        }
    }
}
