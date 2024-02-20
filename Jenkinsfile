pipeline {
    agent {label 'linux'}
    
    parameters {
        string(name: 'branchName', defaultValue: 'main')
        string(name: 'repoUrl', defaultValue: 'https://github.com/NotMe/eShopOnDapr.git')
    }
    
    environment {
        AWS_ECR_REGION = 'AWS_REGION'
        AWS_ECR_REPOSITORY = 'AWS_ECR'
        K8S_NAMESPACE = 'eshop'
        APP_URL = 'DOMAIN_NAME'
    }
    
    stages {
       stage('Clone') {
            steps {
                dir('eShopOnDapr'){
                    sh 'rm -Rf *'
                    git branch: params.branchName, credentialsId: 'github_token', url: params.repoUrl
                    sh 'ls -lah'
                }
            }
        }
        
        stage('Build images') {
            steps {
                dir('eShopOnDapr') {
                    sh '''
                        docker system prune -a -f
                        
                        sed -i "s@localhost@${APP_URL}@g" .env
                        
                        echo " " >> .env
                        echo "REGISTRY="${AWS_ECR_REPOSITORY}"" >> .env
                        
                        cat .env
                        docker compose build
                    '''
                }
            }
        }
        
        stage('Push images to ECR') {
            steps {
                dir('eShopOnDapr') {
                    withAWS(credentials: 'ECR_diploma_user', region: "${AWS_ECR_REGION}") {
                        sh '''
                            aws ecr get-login-password --region "${AWS_ECR_REGION}" | docker login --username AWS --password-stdin "${AWS_ECR_REPOSITORY}"

                            echo '
                            #!/bin/bash
                            set -x
                            for r in $(docker compose convert | grep "image: ${AWS_ECR_REPOSITORY}" | sed -e "s/^.*\\///" |  sed "s/:.*//")
                            do
                              aws ecr create-repository --repository-name "$r"
                            done' >> create-ecr.sh
                        
                            chmod +x create-ecr.sh
                            ./create-ecr.sh
                            rm -rf create-ecr.sh
                        
                            docker compose push
                        '''
                    }
                }
            }
        }
        
        stage('Preparation before deploy') {
            steps {
                dir('eShopOnDapr') {
                    withAWS(credentials: 'ECR_diploma_user', region: "${AWS_ECR_REGION}") {
                        sh '''
                            echo "changing image container registry"
                            
                            sed -i "s@registry: localhost@registry: "${AWS_ECR_REPOSITORY}"@g" ./deploy/k8s/helm/values.yaml
                            
                            echo "changing hostname"
                            
                            sed -i "s@localhost@"${APP_URL}"@g" ./deploy/k8s/helm/values.yaml
                        '''
/*                            echo '
                            #!/bin/bash

                            directory="deploy/k8s/helm/templates"
                            
                            # Check if the target is not a directory
                            if [ ! -d "$directory" ]; then
                              exit 1
                            fi
                            
                            # Loop through files in the target directory
                            for file in "$directory"/*; do
                              if [ -f "$file" ]; then
                                sed -i "s@image: eshopdapr@image: "$AWS_ECR_REPOSITORY"@g" $file
                              fi
                            done' >> prep_before_deploy.sh
                        
                            chmod +x prep_before_deploy.sh
                            ./prep_before_deploy.sh
                            rm -rf prep_before_deploy.sh*/
                    }
                }
            }
        }
        
/*        stage('create EKS cluster') {
            steps {
                withAWS(credentials: 'ECR_diploma_user', region: "${AWS_ECR_REGION}") {
                    sh '''
                        eksctl create cluster -f diploma-k8s.yaml --disable-nodegroup-eviction
                    '''
                }
            }
        }*/
        
        stage('run HELM chart to deploing app to EKS') {
            steps {
                withAWS(credentials: 'ECR_diploma_user', region: "${AWS_ECR_REGION}") {
                    sh '''
                        echo "deploy dapr to the k8s cluster"
                        
                        helm repo add dapr https://dapr.github.io/helm-charts/
                        helm repo add jetstack https://charts.jetstack.io --force-update

                        helm repo update
                        helm upgrade --install dapr dapr/dapr --version=1.12.4 --namespace dapr-system --create-namespace --wait
                        
                        helm upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true
                    
                        cd ./eShopOnDapr/deploy/k8s/helm/
                        helm install eshop .
                        
                        echo "adding secret to k8s"
                        
                        kubectl create secret docker-registry ecr --docker-server=https://AWS@"${AWS_ECR_REGION}" --docker-username=AWS --docker-password=$(aws ecr get-login-password) -n "${K8S_NAMESPACE}"
                    '''
                }
            }
        }
        
/*        stage('install NGINX ingress controller') {
            steps {
                withAWS(credentials: 'ECR_diploma_user', region: "${AWS_ECR_REGION}") {
                    sh '''
                        echo "get the IP address of the cluster load balancer's public endpoint"
                        
                        kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.5/deploy/static/provider/cloud/deploy.yaml
                        
                        kubectl get services ingress-nginx-controller -n ingress-nginx -o=jsonpath='{.status.loadBalancer.ingress[0].ip}'
                    '''
                }
            }
        }*/
        
/*        stage('enable HTTPS access') {
            steps {
                withAWS(credentials: 'ECR_diploma_user', region: "${AWS_ECR_REGION}") {
                    sh '''
                        echo "generating a self-signed certificate, adding it to the k8s secret"
                        
                        openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN="${APP_URL}"/O="${APP_URL}"" -addext "subjectAltName = DNS:"${APP_URL}""
                        
                        kubectl create secret tls eshop-tls --key tls.key --cert tls.crt -n "${K8S_NAMESPACE}"
                        
                        kubectl get ingress -n "${K8S_NAMESPACE}" -o name | xargs -I{} kubectl patch {} -n "${K8S_NAMESPACE}" --type json -p '[{"op": "add", "path": "/spec/tls", "value": [{"secretName": "eshop-tls"}]}]'
                    '''
                }
            }
        }*/
        
    }
}
