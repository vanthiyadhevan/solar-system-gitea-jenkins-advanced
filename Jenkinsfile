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
						reportDir: './',
						reportFiles: 'dependency-check-junit.html',
						reportName: 'dependency-check-HTML Report',
						reportTitles: '', useWrapperFileDirectly: true]) 

						junit allowEmptyResults: true, keepProperties: true, stdioRetention: '', testResults: 'dependency-check-junit.xml'
					}

				}
			}
		}
	}
}