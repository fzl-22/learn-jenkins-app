pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'a7446528-5a71-4c68-a68c-d31c22511740'
        NETLIFY_AUTH_TOKEN = credentials('NETLIFY_AUTH_TOKEN')
        REACT_APP_VERSION = "1.2.$BUILD_ID"
    }

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "[Build Stage]"

                    ls -la
                    node --version
                    npm --version

                    npm ci
                    npm run build

                    ls -la
                '''
            }
        }

        stage('AWS') {
            agent {
                docker {
                    image 'amazon/aws-cli:2.15.53'
                    reuseNode true
                    args "--entrypoint=''"
                }
            }
            environment {
                AWS_S3_BUCKET = "learn-jenkins-app"
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'AWS_USER', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        aws --version

                        aws s3 sync build s3://$AWS_S3_BUCKET
                    '''
                }
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit Test') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            echo "[Unit Test Stage]"

                            test -f build/index.html
                            npm run test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }
                stage('E2E Test') {
                    agent {
                        docker {
                            image 'jenkins-playwright-app'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            echo "[E2E Test Stage]"

                            serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('STAGING DEPLOYMENT') {
            agent {
                docker {
                    image 'jenkins-playwright-app'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "[STAGING DEPLOYMENT]"

                    netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --no-build --json > deploy-output.json
                    export CI_ENVIRONMENT_URL=$(jq -r '.deploy_url' deploy-output.json)

                    echo "[STAGING E2E TESTING]"

                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Staging Report', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        // stage('Approval') {
        //     steps {
        //         timeout(time: 15, unit: 'MINUTES') {
        //             input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!'
        //         }
        //     }
        // }

        stage('PRODUCTION DEPLOYMENT') {
            agent {
                docker {
                    image 'jenkins-playwright-app'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = 'https://golden-frangollo-43e2bc.netlify.app'
            }
            steps {
                sh '''
                    node --version

                    echo "[PRODUCTION DEPLOYMENT]"

                    netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --prod --no-build

                    echo "[PRODUCTION E2E TESTING]"

                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Production Report', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}