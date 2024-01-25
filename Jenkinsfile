pipeline {
	options {
		buildDiscarder(logRotator(numToKeepStr: '8'))
                skipDefaultCheckout() 
                disableConcurrentBuilds() 
		ansiColor('xterm')
	}
	agent any
	environment {
		branch           =       "docker-multiservices"
		repoUrl          =       "https://github.com/candor12/Jenkins-cicd.git"
		gitCreds         =       "gitPAT"
	        ecrRepo          =       "674583976178.dkr.ecr.us-east-2.amazonaws.com/teamimagerepo"
	        dockerImage      =       "${env.ecrRepo}:${env.BUILD_ID}" 
		dockerTag        =       "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}"
		dockerCreds      =       "dockerhubPAT"
	}
	stages{
		stage('SCM Checkout') {
			steps {
				git branch: branch, url: repoUrl, credentialsId: 'gitPAT'
			}
		}
		stage('Build Artifact') {
			steps {
				sh "mvn clean package -DskipTests"
			}
		}
		stage('Docker Image Build') {
			agent { label 'agent1' }
			steps {
				script { 
					cleanWs()
					git branch: branch, url: repoUrl
					sh """
                                        docker compose build 
                                        docker tag teamapp azkabegh/teamapp:${dockerTag} && docker tag teamapp azkabegh/teamapp:latest
				        docker tag teamweb azkabegh/teamweb:${dockerTag} && docker tag teamapp azkabegh/teamweb:latest
                                        docker tag teamdb azkabegh/teamdb:${dockerTag} && docker tag teamapp azkabegh/teamdb:latest
				        """
				}
			}
		}
		stage('Push Image to DockerHub') {
			agent { label 'agent1' }
			steps {
				script {
					def status = sh(returnStatus: true, script: "docker push azkabegh/teamapp:${dockerTag}")
					if (status != 0) {
						sh 'echo $dockerCreds | sudo docker login -u azkabegh --password-stdin'
						sh "docker push azkabegh/teamapp:${dockerTag}"
					}
					sh """
                                        docker push azkabegh/teamapp:latest
				        docker push azkabegh/teamweb:${dockerTag} && docker push azkabegh/teamweb:latest
				        docker push azkabegh/teamdb:${dockerTag} && docker push azkabegh/teamdb:latest
                                        """
			}
		}
	}
		/*stage("Docker Compose Up"){
			steps{
				sh "docker compose up -d"
			}
		}*/
		
	/*	stage('Push Image to ECR') {
			agent { label 'agent1' }
			steps {
				script {
					def status = sh(returnStatus: true, script: 'docker push $dockerImage')
					if (status != 0) {
					    sh "aws ecr get-authorization-token --region us-east-2 --output text --query 'authorizationData[].authorizationToken' | base64 -d | cut -d: -f2 > ecr.txt"
                                            sh 'cat ecr.txt | docker login -u AWS 674583976178.dkr.ecr.us-east-2.amazonaws.com --password-stdin'
					    sh 'docker push $dockerImage'
					}
					sh "docker push ${ecrRepo}:latest"
				}
			}
			post { 
				always {
					sh """ rm -f ecr.txt
					docker rmi -f ${dockerImage}
					docker rmi -f ${ecrRepo}:latest
				        """ 
				}
			}
		}
		stage('EKS Deployment') {
			agent { label 'agent1' }
			when { expression { return params.EksDeploy } }
			steps {
				script { 
					dir('k8s') {
						sh "chmod +x ./cluster.sh && ./cluster.sh" 
						sh '''kubectl apply -f ./eksdeploy.yml
                                                kubectl get deployments && sleep 5 && kubectl get svc
				                '''   
					}
				}
			}
		} */
	} 
	post { always { cleanWs() } }
}
