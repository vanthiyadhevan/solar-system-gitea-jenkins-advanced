pipeline {
	agent any
	tools {
		nodejs nodejs
	}
	stages {
		stage("vm node version") {
			step {
				sh """
					node -v
					npm -v
				"""
			}
		}
	}
}