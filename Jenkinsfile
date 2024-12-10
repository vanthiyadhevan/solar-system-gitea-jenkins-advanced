pipeline {
	agent any
	tools {
		nodejs 'nodejs'
	}
	environment {
		// Test code 
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

						dependencyCheckPublisher failedTotalCritical: 1, pattern: 'dependency-check-report.xml', stopBuild: true
					}

				}
			}
		}
	}
}