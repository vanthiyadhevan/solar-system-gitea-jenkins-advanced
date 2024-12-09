pipeline {
	agent any
	tools {
		nodejs 'nodejs'
	}
	stages {
		stage("vm node version") {
			steps {
				sh """
					node -v
					npm -v
				"""
			}
		}
	}
}