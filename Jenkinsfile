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
		branch           =       "test-tags"
		repoUrl          =       "https://github.com/candor12/jenkins-cicd.git"
		gitCreds         =       "gitPAT"
	        scannerHome      =       tool 'sonartool'
	        ecrRepo          =       "674583976178.dkr.ecr.us-east-2.amazonaws.com/teamimagerepo"
	        dockerImage      =       "${env.ecrRepo}:${env.BUILD_ID}" 
		pullTag          =       "${env.JOB_BASE_NAME}"
	}
	stages{
		stage('SCM Checkout') {
			when { not { buildingTag() } }
			steps {
				cleanWs()
				git branch: branch, url: repoUrl, credentialsId: 'gitPAT'
			}
		} 
		stage('Tag Checkout') {
			when { buildingTag() } 
			steps {
				sh "git clone -c advice.detachedHead=false -b '${pullTag}' --single-branch ${repoUrl}"
			}
		} 
		stage('Build Binaries') {
			when { not { buildingTag() } }
			steps {
				sh "mvn clean package -DskipTests"
			}
		}
		stage('Publish Artifacts') {
			when { not { buildingTag() } }
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
			when { not { buildingTag() } }
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
		stage('Print Tag Name'){
			when { buildingTag() }
			steps{
				script{
					def tag = env.GIT_TAG
					if (tag == null || tag.trim().isEmpty()) {
						error "No tag provided. Deployment aborted."
                    }}}}
		stage('Build Docker Images') {
			when { not { buildingTag() } }
			agent { label 'agent1' }
			steps {
				script { 
					cleanWs()
					git branch: branch, url: repoUrl, credentialsId: 'gitPAT'
					sh '''docker build -t $dockerImage ./
					docker tag $dockerImage $ecrRepo:latest
                                        '''
				}
			}
		}
	} 
}
