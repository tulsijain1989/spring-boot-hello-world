#!/usr/bin/env groovy

// Define DevCloud Artifactory for publishing non-docker image artifacts
def artUploadServer = Artifactory.server('devcloud')

// Change Snapshot to your own DevCloud Artifactory repo name
def Snapshot = 'PROPEL'

pipeline {
    agent none
    environment {
        COMPLIANCEENABLED = true
    }
    options {
		buildDiscarder(logRotator(artifactDaysToKeepStr: '1', artifactNumToKeepStr: '1', daysToKeepStr: '5', numToKeepStr: '10'))    
    }
    stages {
        stage ('Build') {
            agent {
                docker {
                    image 'maven:3.5'
                    label 'dind'
	            args '-v /root/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mvn -B -s settings.xml -DskipTests clean install'
				stash includes: '**/target/*.jar', name: 'artifact'
                stash includes: '**/manifest.yml', name: 'manifest'
            }
            post {
                success {
                    echo "Build stage completed"
                }
                failure {
                    echo "Build stage failed"
                }
            }
        }
        stage ('Unit Tests') {
            agent {
                docker {
                    image 'maven:3.5'
                    label 'dind'
		    args '-v /root/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mvn -B -s settings.xml -Dmaven.test.failure.ignore test'
            }
            post {
                success {
                    script {
                        echo 'STARTING SUREFIRE TEST'
                        sh 'mvn -B -s settings.xml surefire-report:report-only'
                    }
                    echo "Unit Tests stage completed"
                }
                failure {
                    echo "Unit Tests stage failed"
                }
            }
        }
        stage ('Static Code Analysis') {
            agent {
                docker {
                    image 'maven:3.5'
                    label 'dind'
		    args '-v /root/.m2:/root/.m2'
	            
                }
            }
            steps {
                sh 'mvn -B -s settings.xml -e checkstyle:checkstyle pmd:pmd findbugs:findbugs'
            }
            post {
                success {
                    script {
                        step([$class: 'CheckStylePublisher', pattern: '**/target/checkstyle-result.xml'])
                        step([$class: 'FindBugsPublisher', pattern: '**/target/findbugsXml.xml'])
                    }
                    echo "Static Code Analysis stage completed"
                }
                failure {
                    echo "Static Code Analysis stage failed"
                }
            }
        }
        stage ('Documentation') {
            agent {
                docker {
                    image 'maven:3.5'
                    label 'dind'
		    args '-v /root/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mvn -B -s settings.xml -e javadoc:javadoc'
            }
            post {
                success {
                    echo "Documentation stage completed"
                    script {
                        step([$class: 'JavadocArchiver', javadocDir: 'target/site/apidocs', keepAll: false])
                    }
                }
                failure {
                    echo "Documentation stage failed"
                }
            }
        }
        stage ('Code Coverage') {
            agent {
                docker {
                    image 'maven:3.5'
                    label 'dind'
	            args '-v /root/.m2:/root/.m2'
                }
            }
            steps {
                script {
                    step([$class: 'JacocoPublisher', execPattern: '**/target/coverage-reports/jacoco*.exec', exclusionPattern: '**/Messages.class'])
                }
            }
            post {
                success {
                    echo "Code Coverage stage completed"
                }
                failure {
                    echo "Code Coverage stage failed"
                }
            }
        }
        stage ('Publish Artifacts') {
            agent {
                label 'dind'
            }
            steps {
                script {
                    echo 'Publishing Artifacts to Artifactory'                               
                    unstash 'artifact'
                    def uploadSpec = """{
                        "files": [
                            {
                                "pattern": "target/*.jar",
                                "target": "${Snapshot}/predix-cicd-java-sample/"
                            }
                        ]
                    }"""
                    def buildInfo = artUploadServer.upload(uploadSpec)
                    artUploadServer.publishBuildInfo(buildInfo)
                }
            }
            post {
                success {
                    echo "Publish Artifacts stage completed"
                }
                failure {
                    echo "Publish Artifacts stage failed"
                }
            }
        }
        // See README for instructions on what you need to configure for Sonar scan
        stage ('SonarQube Scan') {
            agent {
                docker {
                    image 'maven:3.5'
                    label 'dind'
		    args '-v /root/.m2:/root/.m2'
                }
            }
            steps {
				withSonarQubeEnv('<SonarQube_Name>') {
				// requires SonarQube Scanner for Maven 3.2+
				sh 'mvn -s settings.xml org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar'
				}
			}
            post {
                success {
                    echo "sonarqube-scan stage completed"
                }
                failure {
                    echo "sonarqube-scan stage failed"
                }
            }
        }  
        // See README for instructions on what you need to configure for coverity scan
        stage ('Coverity Scan') {
            agent {             
                docker 
                {
                    image 'registry.gear.ge.com/dig-propel/predixci-coverity8.7.1'
                    label 'dind'
                }                
            }
            environment {
                COVERITY = credentials('Coverity_Token')
            }
            steps {
                sh 'rm -rf $WORKSPACE/covtemp'
                sh 'mkdir -p $WORKSPACE/covtemp'
                sh '/opt/cov-analysis-linux64-8.7.1/bin/cov-build --dir $WORKSPACE/covtemp mvn -B -s settings.xml -Dmaven.test.failure.ignore clean install'
                sh '/opt/cov-analysis-linux64-8.7.1/bin/cov-analyze --dir $WORKSPACE/covtemp'
                sh '/opt/cov-analysis-linux64-8.7.1/bin/cov-commit-defects --dir $WORKSPACE/covtemp --host <Coverity_Host> --https-port 8443 --stream "<Coverity_Stream>" --user "$COVERITY_USR" --password "$COVERITY_PSW"'
            }
            post {
                success {
                    echo "Coverity-Scan stage completed"
                }
                failure {
                    echo "Coverity-Scan stage failed"
                }
            }
        }	    
        // See README for instructions on what you need to configure for Checkmarx scan
	stage ('Checkmarx Scan') {
            agent {                
		docker {
                    image 'registry.gear.ge.com/dig-propel/predixci-checkmarx'
                    label 'dind'
                }          
            }
            environment {
                CHECKMARX = credentials('Checkmarx_ID')
            }
            steps {
                sh '/CxConsolePlugin/runCxConsole.sh scan -v -ProjectName "<Project_Name>" -CxServer https://checkmarx.security.ge.com -CxUser "$CHECKMARX_USR" -CxPassword "$CHECKMARX_PSW" -locationtype folder -locationpath "./" -ReportXML "javascanreport.xml" -ReportPDF "javascanreport.pdf"'
            }
            post {
                success {
                    echo "Checkmarx stage completed"
                }
                failure {
                    echo "Checkmarx stage failed"
                }
            }
        }
        stage ('Deploy') {
            agent {
                docker {
                    image 'registry.gear.ge.com/dig-propel/cf-cli:latest'
                    label 'dind'
                }
            }
            // Enable the 'when' directive for multi-branch pipeline only
            //when {
            //    branch 'master'
            //}
            environment {
                CLOUD_FOUNDRY = credentials('Deployment_Credential')
            }
            steps {
                echo 'Pushing to Cloud Foundry'
                sh "cf login -a https://api.system.aws-usw02-pr.ice.predix.io -u $CLOUD_FOUNDRY_USR -p $CLOUD_FOUNDRY_PSW -o ${env.CF_Org} -s ${env.CF_Space}"
                sh 'cf push'
                sh 'cf a'
            }
            post {
                success {
                    echo "Deploy stage completed"
                }
                failure {
                    echo "Deploy stage failed"
                }
            }
        }
    }
    post {
        success {
            echo 'Your predix-cicd-java pipeline has completed.'
        }
        failure {
            echo "There was an error in the predix-cdcd-java pipeline build"
        }
    }
}
