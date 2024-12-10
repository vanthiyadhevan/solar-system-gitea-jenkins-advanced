pipeline {
	agent any
	tools {
		nodejs 'nodejs'
	}
	stages {
		stage("Installing Dependencies") {
			steps {
				sh 'npm install --no-audit'
			}
		}
		stage("Dependency Scanning") {
			parallel {
				stage("NPM Dependency Audit") {
					steps {
						sh 'npm audit --audit-level=critical'
					}
				}
				stage("OWASP Dependency Check") {
					steps {
						dependencyCheck additionalArguments: '''
							--scan ./
							--out ./
							--format ALL
							--prettyPrint
						''', odcInstallation: 'OWASP-DepCheck-10'

						dependencyCheckPublisher pattern: 'dependency-check-report.xml', stopBuild: true

						publishHTML([allowMissing: true,
						alwaysLinkToLastBuild: true, keepAll: true,
						reportDir: './', reportFiles: 'index.html',
						reportName: 'dependency-check-HTML Report',
						reportTitles: 'dependency-check-jenkins.html',
						useWrapperFileDirectly: true]) 

						junit allowEmptyResults: true, keepProperties: true, stdioRetention: '', testResults: 'dependency-check-junit.xml'
					}

				}
			}
		}
	}
}