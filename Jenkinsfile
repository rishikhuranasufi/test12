#!groovy

pipeline {
	agent any 

	options{
		skipDefaultCheckout(true)
	}

	stages {
	stage ("Checkout") {
		steps{
			checkout([$class: 'GitSCM',branches: [[name: '*/master']],doGenerateSubmoduleConfigurations: false,
			extensions: [],submoduleCfg: [],userRemoteConfigs: [[credentialsId: "test",
			url: "git@github.com:rishikhuranasufi/test12.git"]]])

			load "jenkins-env.properties"
		}
	}

	stage("Build & Unit Tests") {
		steps{
		script{
			def mvnHome = tool 'M3';
			sh "${mvnHome}/bin/mvn ${build_with_unittest}"

		}
		}
		}

	stage("SonarQube") {
		steps{
		script{
			def scannerHome = tool 'sonarqubescanner';

			withSonarQubeEnv('Sonar Server') {
			sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=${sonar_projectKey} -Dsonar.projectName=${sonar_projectName} -Dsonar.sources=${sonar_sources}  -Dsonar.java.binaries=${sonar_java_maven_binaries} "

			sh "cat .scannerwork/report-task.txt"
			def props = readProperties  file: '.scannerwork/report-task.txt'
			echo "properties=${props}"
			def sonarServerUrl=props['serverUrl']
			def ceTaskUrl= props['ceTaskUrl']

			def ceTask

			timeout(time: 1, unit: 'MINUTES') {
				waitUntil {
				def response = httpRequest ceTaskUrl
				ceTask = readJSON text: response.content
				echo ceTask.toString()
				return "SUCCESS".equals(ceTask["task"]["status"])
				}
			}

			def response2 = httpRequest url : sonarServerUrl + "/api/qualitygates/project_status?analysisId=" + ceTask["task"]["analysisId"], authentication: 'sonar-quality-auth-credentials-id'
			def qualitygate =  readJSON text: response2.content
			echo qualitygate.toString()

			if ("ERROR".equals(qualitygate["projectStatus"]["status"])) {
			error  "Quality Gate failure"
			}

			}
		}
		}
		}

	stage("Deploy to Nexus") {
		steps{
		script{
			def mvnHome = tool 'M3';
			sh "${mvnHome}/bin/mvn clean install deploy"
		}
		}
		}

	stage("SourceClear") {
		steps{
		script{
			sh "export PATH=${PATH}:/opt/maven/bin && export SRCCLR_API_TOKEN=${srcclr_api_token} && /opt/srcclr-3.0.18/bin/srcclr scan . --debug" 
		}
		}
		}

	stage("Coverity") {
		steps{
		script{
			def mvnHome = tool 'M3';
			sh "/opt/coverity/bin/cov-build --dir ./idir --fs-capture-search . ${mvnHome}/bin/mvn clean install -DskipTests=true" 
			sh "/opt/coverity/bin/cov-analyze --dir ./idir --all --webapp-security --enable-virtual --enable-fnptr --enable-constraint-fpp" 
			//sh "/opt/coverity/bin/cov-commit-defects --dir ./idir --stream ${cov_project_stream} --user ${cov_connect_user}  --password ${cov_connect_password} --host ${coverity_connect_host} --https-port ${cov_connect_port} --on-new-cert trust" 
		}
		}
		}
	stage("dev_deploy") {
		steps{
		script{
			sh "cf api ${dev_cf_api_endpoint}"
			withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "${dev_cf_credentials_id}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
				sh "cf auth $USERNAME $PASSWORD"
			}
			sh "cf target -o ${dev_cf_org} -s ${dev_cf_space}"
			sh "cf create-service newrelic standard newrelic"
			sh "cf push -f ${dev_cf_manifest}"
			sh "cf logout"
		}
		}
		}


	}


}
