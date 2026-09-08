pipeline {
    agent any
	
	environment {
         DOCKERHUB_REPO = "svbagade24/microservice-kubernetes-demo"
         IMAGE_TAG      = "${BUILD_NUMBER}"
         MANIFEST_REPO  = "https://github.com/s-v-bagade/k8s-manifests-microapp.git"
    }
	
	stages {
	   
	   stage('Docker Build') {
          steps {
             dir('microservice-kubernetes-demo') {
                sh """
                docker build -f microservice-kubernetes-demo-order/Dockerfile \
                    -t ${DOCKERHUB_REPO}-order:${IMAGE_TAG} .
				docker tag ${DOCKERHUB_REPO}-order:${IMAGE_TAG} ${DOCKERHUB_REPO}-order:latest
				
                docker build -f microservice-kubernetes-demo-customer/Dockerfile \
                    -t ${DOCKERHUB_REPO}-customer:${IMAGE_TAG} .
				docker tag ${DOCKERHUB_REPO}-customer:${IMAGE_TAG} ${DOCKERHUB_REPO}-customer:latest
				
                docker build -f microservice-kubernetes-demo-catalog/Dockerfile \
                    -t ${DOCKERHUB_REPO}-catalog:${IMAGE_TAG} .
				docker tag ${DOCKERHUB_REPO}-catalog:${IMAGE_TAG} ${DOCKERHUB_REPO}-catalog:latest
                """
             }
            }
        }
		
	   stage ('Push to Docker Hub') {
	        steps {
			   withCredentials ([usernamePassword(
			       credentialsId: 'dockerhub-creds',
				   usernameVariable: 'Docker_USER',
				   passwordVariable: 'DOCKER_PASS'
				)]) {
				    sh """
					    echo \$DOCKER_PASS | docker login -u \$Docker_USER --password-stdin
						docker push ${DOCKERHUB_REPO}-order:${IMAGE_TAG}
						docker push ${DOCKERHUB_REPO}-order:latest
						
						docker push ${DOCKERHUB_REPO}-customer:${IMAGE_TAG}
						docker push ${DOCKERHUB_REPO}-customer:latest
						
						docker push ${DOCKERHUB_REPO}-catalog:${IMAGE_TAG} 
						docker push ${DOCKERHUB_REPO}-catalog:latest
					"""
				}
			}
	   }
	   
	   stage ('Checkout Manifest Repo') {
	        steps {
			    dir('manifests') {
				    git branch: 'master',
					url: "${MANIFEST_REPO}",
                    credentialsId: 'github-cred'
				}
			}
	   }
	   
	   stage('Deploy to Kubernetes') {
            steps {
               withCredentials([file(
			     credentialsId: 'kubeconfig-cred-id', 
				 variable: 'KUBECONFIG'
				 )]) {
                   sh """
                     sed -i "s|__IMAGE_TAG__|${IMAGE_TAG}|g" manifests/values.yaml

                     helm upgrade --install microapp manifests/ \
                     -n default \
                     --atomic \
                     --timeout 180s
                   """
                   }
            }
        }
           
        }
	   
	   post {
         success {
            echo "Deployment successful — release microapp updated with image tag ${IMAGE_TAG}"
         }
         failure {
            echo "Pipeline failed — Helm automatically rolled back via --atomic"
         }
        }
	


    }
