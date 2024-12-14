pipeline {
	agent any
	tools {
		nodejs 'nodejs'
	}
	environment {
  		MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
  		MONGO_USERNAME = credentials('mongo-db-creds')
  		MONGO_PASSWORD = credentials('mongo-db-creds')
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
						reportFiles: 'dependency-check-jenkins.html',
						reportName: 'dependency-check-HTML Report',
						reportTitles: '', useWrapperFileDirectly: true])

						junit allowEmptyResults: true, stdioRetention: '', testResults: 'dependency-check-junit.xml'

					}
				}
			}
		}
		stage("Unit Testing") {
					steps {
						withCredentials([usernamePassword(credentialsId: 'mongo-db-creds', passwordVariable: 'MONGO_PASSWORD', usernameVariable: 'MONGO_USERNAME')]) {
    						// some block
    						sh 'npm test'
						}

					junit allowEmptyResults: true, stdioRetention: '', testResults: 'test-results.xml'	
				}
		}
		stage("Code Coverage") {
				steps {
					withCredentials([usernamePassword(credentialsId: 'mongo-db-creds', passwordVariable: 'MONGO_PASSWORD', usernameVariable: 'MONGO_USERNAME')]) {
    				catchError(buildResult: 'SUCCESS', message: 'Oops! it will fixed in future release', stageResult: 'UNSTABLE') {
    					sh 'npm run coverage'
					}
				}
				publishHTML([allowMissing: true,
						alwaysLinkToLastBuild: true, keepAll: true,
						reportDir: 'coverage/lcov-report',
						reportFiles: 'index.html',
						reportName: 'Code-Coverage-HTML Report',
						reportTitles: '', useWrapperFileDirectly: true])
			}
		}
	}
}