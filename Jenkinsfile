pipeline {
	options {
		buildDiscarder(logRotator(numToKeepStr: '10'))
                skipDefaultCheckout() 
                disableConcurrentBuilds() 
	}
	agent any
	parameters {
		booleanParam(name: "EksDeploy", defaultValue: false, description: "Deploy the Build to EKS Cluster")
		booleanParam(name: "AnsibleDeploy", defaultValue: false, description: "Deploy the Build to Target Server using Ansible")
		booleanParam(name: "Scan", defaultValue: false, description: "By Pass SonarQube, Grype and Trivy Scan")
	}
	environment {
		branch           =       "master"
		repoUrl          =       "https://github.com/candor12/jenkins-cicd.git"
		gitCreds         =       "gitPAT"
	        scannerHome      =       tool 'sonartool'
	        ecrRepo          =       "674583976178.dkr.ecr.us-east-2.amazonaws.com/teamimagerepo"
	        dockerImage      =       "${env.ecrRepo}:${env.BUILD_ID}" 
	}
	stages{
		stage('SCM Checkout') {
			steps {
				git branch: branch, url: repoUrl, credentialsId: 'gitPAT'
			}
		} 
		stage('Build Binaries') {
			steps {
				sh "mvn clean package -DskipTests"
			}
		}
		stage('JUnit Test') {
			tools { jdk "jdk-11" }
			steps {
				sh "mvn test"
			}
		}
		stage('SonarQube Scan') {
			when { not { expression { return params.Scan  } } }
			steps {
				script { 
					withSonarQubeEnv('sonar') {
						sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=jenkins1 \
                                                -Dsonar.projectName=jenkins1 \
                                                -Dsonar.projectVersion=1.0 \
                                                -Dsonar.sources=src/ \
                                                -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                                                -Dsonar.junit.reportsPath=target/surefire-reports/ \
                                                -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                                                -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
					}
					echo "Waiting for Quality Gate"
					timeout(time: 5, unit: 'MINUTES') {
						def qualitygate = waitForQualityGate(webhookSecretId: 'sonarhook')
						if (qualitygate.status != "OK") { 
							catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') { 
								sh "exit 1"  
							}
						}
					}
				}
			}
		} 
		stage('Publish Artifacts') {
			steps {
				script {
					sh "mvn deploy -DskipTests -Dmaven.install.skip=true | tee nexus.log"
					def artifactUrl     =     sh(returnStdout: true, script: 'tail -20 nexus.log | grep ".war" nexus.log | grep -v INFO | grep -v Uploaded')
				        nexusArtifact       =     artifactUrl.drop(20)    
                                        tag1                =     nexusArtifact.drop(98)
					tag3                =     tag1.replaceAll(".war", "") 
					echo "Artifact URL: ${nexusArtifact}"
				}
			}
		}
		stage('Push Tag to Repository') {
			steps { 
				withCredentials([usernamePassword(credentialsId: 'gitPAT',usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
					script{
						def pomVersion     =  sh(returnStdout: true, script: "mvn -DskipTests help:evaluate -Dexpression=project.version -q -DforceStdout")
						gitTag             =  "${pomVersion}${tag3}"
						sh "git tag $gitTag"
                                                sh "git push origin $gitTag"
					}
				}
			}
		} 
		stage('Build Docker Images') {
			agent { label 'agent1' }
			steps {
				script { 
					git branch: branch, url: repoUrl, credentialsId: 'gitPAT'
					sh '''docker build -t $dockerImage ./
					docker tag $dockerImage $ecrRepo:latest
                                        '''
				}
			}
		}
		stage ('Grype Image Scan') {
			agent { label 'agent1' }
			when { not { expression { return params.Scan  } } }
			steps {
				script {
					curl -sfL https://github.com/candor12/templates/blob/main/grypehtml.tmpl > ./grypehtml.tpl
					sh "grype ${dockerImage} --scope all-layers --fail-on critical -o template -t ./grypehtml.tmpl > ./grype.html"
				}
			}
			post { always { 
				archiveArtifacts artifacts: "grype.html", fingerprint: true
				publishHTML target : [allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true,
						      reportDir: './', reportFiles: 'grype.html', reportName: 'Grype Scan Report', reportTitles: 'Grype Scan Report']
				      }
			     }
		}
		stage ('Trivy Image Scan') {
			agent { label 'agent1' }
			when { not { expression { return params.Scan  } } }
			steps {
				script {
					 sh 'curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl > ./html.tpl'
				         sh 'trivy image --skip-db-update --skip-java-db-update --cache-dir ~/trivy/ --format template --template \"@./html.tpl\" -o trivy.html --severity MEDIUM,HIGH,CRITICAL ${dockerImage}' 
				}
			}
			post { always { archiveArtifacts artifacts: "trivy.html", fingerprint: true
				                     publishHTML target : [allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true,
									   reportDir: './', reportFiles: 'trivy.html', reportName: 'Trivy Scan', reportTitles: 'Trivy Scan']
				      }
			     }
		}		
		stage('Push Image to ECR') {
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
		stage('Fetch from Nexus & Deploy using Ansible') {
			agent { label 'agent1' }
			when { expression { return params.AnsibleDeploy } }
			steps {
				script{ 
					dir('ansible') {
					sh "ansible-playbook deployment.yml -e NEXUS_ARTIFACT=${nexusArtifact}"
					}
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
		} 
	} 
	post { always { cleanWs() } }
}
