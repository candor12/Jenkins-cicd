pipeline {
	options {
		buildDiscarder(logRotator(numToKeepStr: '8'))
                skipDefaultCheckout() 
	}
	agent any
	environment {
		branch           =       "trial"
		repoUrl          =       "https://github.com/candor12/jenkins-cicd.git"
		gitCreds         =       "gitPAT"
	        scannerHome      =       tool 'sonartool'
	        ecrRepo          =       "674583976178.dkr.ecr.us-east-2.amazonaws.com/teamimagerepo"
	       
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
		
		stage('Publish Artifact to Nexus') {
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
						dockerImage        =       "${env.ecrRepo}:${gitTag}" 
					}
				}
			}
		} 
		stage('Docker Image Build') {
			agent { label 'agent1' }
			steps {
				script { 
					cleanWs()
					git branch: branch, url: repoUrl
					sh '''docker build -t ${dockerImage} ./
					docker tag $dockerImage $ecrRepo:latest
                                        '''
				}
			}
		}
		
	} 
	post { always { cleanWs() } }
}
