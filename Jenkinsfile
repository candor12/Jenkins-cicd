pipeline {
	options {
		buildDiscarder(logRotator(numToKeepStr: '8'))
                skipDefaultCheckout() 
                disableConcurrentBuilds() 
		ansiColor('xterm')
	}
	agent any
	parameters {
		booleanParam(name: "EksDeploy", defaultValue: false, description: "Deploy the Build to EKS Cluster")
		booleanParam(name: "Scan", defaultValue: false, description: "By Pass SonarQube and Grype Scan")
	}
	environment {
		branch           =       "buildingTag"
		repoUrl          =       "https://github.com/candor12/jenkins-cicd.git"
		gitCreds         =       "gitPAT"
	        scannerHome      =       tool 'sonartool'
	        ecrRepo          =       "674583976178.dkr.ecr.us-east-2.amazonaws.com/teamimagerepo"
	        dockerImage      =       "${env.ecrRepo}:${env.BUILD_ID}-${env.BUILD_TIMESTAMP}" 
	}
	stages{
		stage('SCM Checkout') {
			steps {
				git branch: branch, url: repoUrl, credentialsId: 'gitPAT'
			}
		}
		stage('Build Artifact') {
			when { not { buildingTag() } }
			steps {
				sh "mvn clean package -DskipTests"
			}
		}
		stage('JUnit Test'){
			when { not { buildingTag() } }
			//jdk-17 fails this stage.
			tools { jdk "jdk-11" }
			steps {
				sh "mvn test"
			}
			post {
				always {
					junit(testResults: '**/surefire-reports/*.xml', allowEmptyResults : true)
				}
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
		stage('Publish Artifact to JFrog') {
			when { not { buildingTag() } }
			steps {
				script {
					sh "mvn deploy -DskipTests -Dmaven.install.skip=true | tee jfrog.log"
					def artifactUrl      =     sh(returnStdout: true, script: 'tail -20 jfrog.log | grep ".war" jfrog.log | grep -v INFO | grep -v Uploaded')
				        jfrog_Artifact       =     artifactUrl.drop(20)  
					def tag1             =     jfrog_Artifact.drop(101)
				        tag2                 =     tag1.take(16) 
					echo "Artifact URL: ${jfrog_Artifact}"
				}
			}
		}
		stage('Push Tag to Repository') {
			when { not { buildingTag() } }
			steps { 
				withCredentials([usernamePassword(credentialsId: 'gitPAT',usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
					script{
					        def pomVersion  =  sh(returnStdout: true, script: "mvn -DskipTests help:evaluate -Dexpression=project.version -q -DforceStdout")
						gitTag          =  "${pomVersion}${tag2}-${BUILD_ID}"
						sh """git tag -a ${gitTag} -m 'Pushed by Jenkins'
                                                git push ${repoUrl} --tags
				                """
					}
				}
			}
		} 
		stage('Docker Image Build') {
			agent { label 'agent1' }
			when { not { buildingTag() } }
			steps {
				script { 
					cleanWs()
					git branch: branch, url: repoUrl
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
					sh "grype ${dockerImage} --scope all-layers --fail-on critical -o template -t ~/jenkins/grype/html.tmpl > ./grype.html"
				}
			}
			post { always { 
				archiveArtifacts artifacts: "grype.html", fingerprint: true
				publishHTML target : [allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true,
						      reportDir: './', reportFiles: 'grype.html', reportName: 'Grype Scan Report', reportTitles: 'Grype Scan Report']
				      }
			     }
			}
		stage('Push Image to ECR') {
			agent { label 'agent1' }
			when { not { buildingTag() } }
			steps {
				script {
					def status = sh(returnStatus: true, script: 'docker push $dockerImage')
					if (status != 0) {
					    sh "aws ecr get-authorization-token --region us-east-2 --output text --query 'authorizationData[].authorizationToken' | base64 -d | cut -d: -f2 > ecr.txt"
                                            sh 'cat ecr.txt | docker login -u AWS 674583976178.dkr.ecr.us-east-2.amazonaws.com --password-stdin'
					    sh 'docker push $dockerImage'
					    sh 'rm -f ecr.txt'
					}
					sh "docker push ${ecrRepo}:latest"
				}
			}
			post { 
				always {
					sh """docker rmi -f ${dockerImage}
					docker rmi -f ${ecrRepo}:latest""" 
				}
			}
		}
		stage('EKS Deployment') {
			agent { label 'agent1' }
			when { expression { return params.EksDeploy } }
			steps {
				script { 
					dir('k8s') {
						sh "./cluster.sh" 
						sh '''kubectl apply -f ./eksdeploy.yml
                                                sleep 6 && kubectl get all
				                '''   
					}
				}
			}
		} 
	} 
	post { 
		always {
			cleanWs() 
		} 
	}
}
