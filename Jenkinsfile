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
	        dockerImage      =       "${env.ecrRepo}:${gitTag}" 
		pullTag          =       "${env.JOB_BASE_NAME}"
	}
	stages{
		stage('SCM Checkout') {
			when { not { buildingTag() } }
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
			tools { jdk "jdk-11" }
			steps {
				sh "mvn test"
			}
		}
		stage('Pull Tag and Artifact') {
			when { buildingTag() }
			steps {
				sh "git clone -c advice.detachedHead=false -b '${pullTag}' --single-branch ${repoUrl}"
			}
		}
		stage('SonarQube Scan') {
			when { allOf { 
					not { buildingTag() }
					not { expression { return params.Scan  } } } }
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
                                                -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml''' }
					echo "Waiting for Quality Gate"
					timeout(time: 5, unit: 'MINUTES') {
						def qualitygate = waitForQualityGate(webhookSecretId: 'sonarhook')
						if (qualitygate.status != "OK") { 
							catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') { 
								sh "exit 1"  } } }
				}
			}
		} 
		stage('Publish Artifacts to JFrog') {
			when { not { buildingTag() } }
			steps {
				script {
					sh "mvn deploy -DskipTests -Dmaven.install.skip=true | tee jfrog.log"
					def artifactUrl      =     sh(returnStdout: true, script: 'tail -20 jfrog.log | grep ".war" jfrog.log | grep -v INFO | grep -v Uploaded')
				        jfrog_Artifact       =     artifactUrl.drop(20)
					echo "Artifact URL: ${jfrog_Artifact}"
				}
			}
		}
		stage('Push Tag to Repository') {
			when { not { buildingTag() } }
			steps { 
				withCredentials([usernamePassword(credentialsId: 'gitPAT',usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
					script{
						def tag          =  sh(returnStdout: true, script: """echo "${jfrog_Artifact}" | sed 's/.*-\\([0-9.]*-[0-9]*\\).*/\\1/' """)
					        def pomVersion   =  sh(returnStdout: true, script: "mvn -DskipTests help:evaluate -Dexpression=project.version -q -DforceStdout")
						gitTag           =  "${pomVersion}${tag}"
						sh """git tag ${gitTag}
                                                git push ${repoUrl} --tags"""
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
				        def dockerTag = "${gitTag}"
					dockerImg     = "${ecrRepo}:${dockerTag}"
					sh """docker build -t ${dockerImg} ./"""
					sh 'docker tag ${dockerImg} $ecrRepo:latest'
				}
			}
		}
		stage ('Grype Image Scan') {
			agent { label 'agent1' }
			when { not { expression { return params.Scan  } } }
			steps {
				script {
					sh "grype ${dockerImg} --scope all-layers --fail-on critical -o template -t ~/jenkins/grype/html.tmpl > ./grype.html"
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
					def status = sh(returnStatus: true, script: 'docker push ${dockerImg}')
					if (status != 0) {
					    sh "aws ecr get-authorization-token --region us-east-2 --output text --query 'authorizationData[].authorizationToken' | base64 -d | cut -d: -f2 > ecr.txt"
                                            sh 'cat ecr.txt | docker login -u AWS 674583976178.dkr.ecr.us-east-2.amazonaws.com --password-stdin'
					    sh "docker push ${dockerImg}"
					    sh 'rm -f ecr.txt'
					}
					sh "docker push ${ecrRepo}:latest"
				}
			}
			post { 
				always {
					sh """docker rmi -f ${dockerImg}
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
