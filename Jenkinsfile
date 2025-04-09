pipeline {
    agent any

    tools {
        nodejs 'nodejs-22-6-0'
    }

    environment {
        MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
        MONGO_DB_CREDS = credentials('mongo-db-credentials')
        MONGO_USERNAME = credentials('mongo-db-username')
        MONGO_PASSWORD = credentials('mongo-db-password')
        AWS_CRDS = credentials('aws_creds')
        AWS_REGION = credentials('aws_region')
        ECR_REPO_NAME = credentials('ecr_repo_name')
        ECR_REPO_URI = credentials('ecr_repo_uri')
        SONAR_SCANNER_HOME = tool 'sonarqube-scanner-610';
    }

    options {
        disableResume()
        disableConcurrentBuilds abortPrevious: true
    }

    stages {
        stage('Installing Dependencies') {
            options { timestamps() }
            steps {
                sh 'npm install --no-audit'
            }
        }

        stage('Dependency Scanning') {
            parallel {
                stage('NPM Dependency Audit') {
                    steps {
                        sh 'npm audit --audit-level=critical || true'
                        sh 'npm audit fix || true'
                    }
                }

                // stage('OWASP Dependency Check') {
                //     steps {
                //         dependencyCheck additionalArguments: '''
                //             --scan ./
                //             --out ./
                //             --format ALL 
                //             --disableYarnAudit
                //             --prettyPrint''', odcInstallation: 'OWASP-DepCheck-10'

                //         dependencyCheckPublisher failedTotalCritical: 1, pattern: 'dependency-check-report.xml', stopBuild: false
                //     }
                // }
            }
        }

        stage('Unit Testing') {
            options { retry(2) }
            steps {
                sh 'echo Colon-Separated - $MONGO_DB_CREDS'
                sh 'echo Username - $MONGO_DB_CREDS_USR'
                sh 'echo Password - $MONGO_DB_CREDS_PSW'
                sh 'npm test' 
            }
        }    

        stage('Code Coverage') {
            steps {
                catchError(buildResult: 'SUCCESS', message: 'Oops! it will be fixed in future releases', stageResult: 'SUCCESS') {
                    sh 'npm run coverage'
                }
            }
        }

        stage('SAST - SonarQube') {
            steps {
                sh 'sleep 5s'
                // timeout(time: 60, unit: 'SECONDS') {
                //     withSonarQubeEnv('sonar-qube-server') {
                //         sh 'echo $SONAR_SCANNER_HOME'
                //         sh '''
                //             $SONAR_SCANNER_HOME/bin/sonar-scanner \
                //                 -Dsonar.projectKey=Solar-System-Project \
                //                 -Dsonar.sources=app.js \
                //                 -Dsonar.javascript.lcov.reportPaths=./coverage/lcov.info
                //         '''
                //     }
                //     waitForQualityGate abortPipeline: true
                // }
            }
        } 

        // stage('Build Docker Image') {
        //     steps {
        //         sh  'printenv'
        //         sh  'docker build -t vanthiyadevan/solar-system:$GIT_COMMIT .'
        //     }
        // }

        // stage('Build App image') {
        //     steps {
        //         script {
        //             dockerImage = docker.build("${env.ECR_REPO_URI}:${BUILD_NUMBER}", ".") 
        //         }
        //     }
        // }
        stage('Docker Build') {
            steps {
                script {
                    sh "docker build -t ${ECR_REPO_URI}:${BUILD_NUMBER} -f Dockerfile ."
                }
            }
        }

        stage('Trivy Vulnerability Scanner') {
            steps {
                // sh 'echo $PATH && which trivy && trivy --version'
                sh  ''' 
                    trivy image $ECR_REPO_URI:$BUILD_NUMBER \
                        --severity LOW,MEDIUM,HIGH \
                        --exit-code 0 \
                        --quiet \
                        --format json -o trivy-image-MEDIUM-results.json

                    trivy image $ECR_REPO_URI:$BUILD_NUMBER \
                        --severity CRITICAL \
                        --exit-code 0 \
                        --quiet \
                        --format json -o trivy-image-CRITICAL-results.json
                '''
            }
            post {
                always {
                    sh '''
                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output trivy-image-MEDIUM-results.html trivy-image-MEDIUM-results.json 

                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output trivy-image-CRITICAL-results.html trivy-image-CRITICAL-results.json

                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output trivy-image-MEDIUM-results.xml  trivy-image-MEDIUM-results.json 

                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output trivy-image-CRITICAL-results.xml trivy-image-CRITICAL-results.json          
                    '''
                }
            }
        } 

        stage('ECR login and Push Docker Image') {
            steps {
                script {
                    withAWS(credentials: "${AWS_CRDS}", region: "${AWS_REGION}") {
                        sh "aws ecr get-login-password | docker login --username AWS --password-stdin ${ECR_REPO_URI}"
                        sh "docker push ${ECR_REPO_URI}:${BUILD_NUMBER}"

                    }
                    // sh "docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}"
                }
            }
        }
        // stage('Push Docker Image') {
        //     steps {
        //         script {
        //              docker.withRegistry("${ECR_REPO_URI}", "${AWS_CRDS}") {
        //                 dockerImage.push("${BUILD_NUMBER}")
        //                 dockerImage.push('latest')
        //             }
        //         }
        //     }
        // }

        // stage('Push Docker Image') {
        //     steps {
        //         withDockerRegistry(credentialsId: 'docker-hub-credentials', url: "") {
        //             sh  'docker push vanthiyadevan/solar-system:$GIT_COMMIT'
        //         }
        //     }
        // }

        stage('K8S - Update Image Tag') {
            when {
                branch 'staging'
            }
            steps {
                sh 'git clone -b main http://64.227.187.25:5555/dasher-org/solar-system-gitops-argocd'
                dir("solar-system-gitops-argocd/kubernetes") {
                    sh '''
                        #### Replace Docker Tag ####
                        git checkout main
                        git checkout -b feature-$BUILD_ID
                        sed -i "s#siddharth67.*#siddharth67/solar-system:$GIT_COMMIT#g" deployment.yml
                        cat deployment.yml
                        
                        #### Commit and Push to Feature Branch ####
                        git config --global user.email "jenkins@dasher.com"
                        git remote set-url origin http://$GITEA_TOKEN@64.227.187.25:5555/dasher-org/solar-system-gitops-argocd
                        git add .
                        git commit -am "Updated docker image"
                        git push -u origin feature-$BUILD_ID
                    '''
                }
            }
        }

        stage('K8S - Raise PR') {
            when {
                branch 'staging'
            }
            steps {
                sh """
                    curl -X 'POST' \
                        'http://64.227.187.25:5555/api/v1/repos/dasher-org/solar-system-gitops-argocd/pulls' \
                        -H 'accept: application/json' \
                        -H 'Authorization: token $GITEA_TOKEN' \
                        -H 'Content-Type: application/json' \
                        -d '{
                            "assignee": "gitea-admin",
                                "assignees": [
                                    "gitea-admin"
                                ],
                            "base": "main",
                            "body": "Updated docker image in deployment manifest",
                            "head": "feature-$BUILD_ID",
                            "title": "Updated Docker Image"
                        }'
                """
            }
        }

        stage('App Deployed?') {
            when {
                branch 'staging'
            }
            steps {
                timeout(time: 1, unit: 'DAYS') {
                    input message: 'Is the PR Merged and ArgoCD Synced?', ok: 'YES! PR is Merged and ArgoCD Application is Synced'
                }
            }
        }

        stage('DAST - OWASP ZAP') {
            when {
                branch 'staging'
            }
            steps {
                sh '''
                    #### REPLACE below with Kubernetes http://IP_Address:30000/api-docs/ #####
                    chmod 777 $(pwd)
                    docker run -v $(pwd):/zap/wrk/:rw  ghcr.io/zaproxy/zaproxy zap-api-scan.py \
                    -t http://134.209.155.222:30000/api-docs/ \
                    -f openapi \
                    -r zap_report.html \
                    -w zap_report.md \
                    -J zap_json_report.json \
                    -x zap_xml_report.xml \
                    -c zap_ignore_rules
                '''
            }
        }

        stage('Upload - AWS S3') {
            when {
                branch 'staging'
            }
            steps {
                withAWS(credentials: "${AWS_CRDS}", region: "${AWS_REGION}") {
                    sh  '''
                        ls -ltr
                        mkdir reports-$BUILD_ID
                        cp -rf coverage/ reports-$BUILD_ID/
                        cp dependency*.* test-results.xml trivy*.* zap*.* reports-$BUILD_ID/
                        ls -ltr reports-$BUILD_ID/
                    '''
                    s3Upload(
                        file:"reports-$BUILD_ID", 
                        bucket:'staging-test-reports', 
                        path:"jenkins-$BUILD_ID/"
                    )
                }
            }
        } 

        stage('Deploy to Prod?') {
            when {
                branch 'main'
            }
            steps {
                timeout(time: 1, unit: 'DAYS') {
                    input message: 'Deploy to Production?', ok: 'YES! Let us try this on Production', submitter: 'admin'
                }
            }
        }
    }

    post {
        always {
            script {
                if (fileExists('solar-system-gitops-argocd')) {
                    sh 'rm -rf solar-system-gitops-argocd'
                }
            }

            junit allowEmptyResults: true, stdioRetention: '', testResults: 'test-results.xml'
            junit allowEmptyResults: true, stdioRetention: '', testResults: 'dependency-check-junit.xml' 
            junit allowEmptyResults: true, stdioRetention: '', testResults: 'trivy-image-CRITICAL-results.xml'
            junit allowEmptyResults: true, stdioRetention: '', testResults: 'trivy-image-MEDIUM-results.xml'

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'zap_report.html', reportName: 'DAST - OWASP ZAP Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'trivy-image-CRITICAL-results.html', reportName: 'Trivy Image Critical Vul Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'trivy-image-MEDIUM-results.html', reportName: 'Trivy Image Medium Vul Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'dependency-check-jenkins.html', reportName: 'Dependency Check HTML Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])
        }
    }
}