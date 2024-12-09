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
			}

		}
	}
}