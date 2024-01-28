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
				checkout scm: [$class: 'GitSCM', userRemoteConfigs: [[url: repoUrl, credentialsId: 'gitPAT' ]], branches: [[name: pullTag]]], poll: false
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
					echo "${pullTag}"
					def snapshotRepo    =   "http://172.31.17.3:8081/repository/maven-snapshots"
					def releaseRepo     =   "http://172.31.17.3:8081/repository/maven-releases"
					def group           =   sh(returnStdout: true, script: "mvn -DskipTests help:evaluate -Dexpression=project.groupId -q -DforceStdout")
					groupId             =   group.replaceAll('/','.')
					def artifact        =   sh(returnStdout: true, script: "mvn -DskipTests help:evaluate -Dexpression=project.artifactId -q -DforceStdout")
					artifactId          =   artifact.replaceAll('/','.')
				        def artifact        =   "${pullTag}"
					if (artifact.contains("SNAPSHOT")) {
						artifactURL =   "${snapshotRepo}/${groupId}/${artifactId}/${pomVersion}/${artifactId}-v2.3-20240128.093751-12.war"
					}
                    }}}
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
