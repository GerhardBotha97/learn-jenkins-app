pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '5ad08bd9-02fe-4ef0-b55a-cb6630ea5b20'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
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
                echo 'INIT BUILD...'
                sh '''
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
                echo "BUILD DONE..."
            }
        }

        stage('Run Tests') {
            parallel {
                stage('Test') {
                
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }

                    steps {
                        echo 'RUN TEST...'
                        sh '''
                            test -f build/index.html
                            npm test
                        '''
                    }

                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }

                    steps {
                        echo 'RUN TEST...'
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }

                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy STAGING') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
        
            steps {
                echo 'DEPLOYING TO STAGING...'
                sh '''
                    npm install netlify-cli node-jq
                    node_modules/.bin/netlify --version
                    echo "Delploying to test SITE ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --json > staging.json
                    node_modules/.bin/node-jq -r '.deploy_url' staging.json
                '''
            }

            script {
                env.STAGE_URL = sh(script: "node_modules/.bin/node-jq -r '.deploy_url' staging.json", returnStdout: true)
            }
        }

        stage('Staging E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            
            environment {
                CI_ENVIRONMENT_URL = ${env.STAGE_URL}
            }
            
            steps {
                echo 'RUN TEST - STAGING...'
                sh '''
                    npx playwright test --reporter=html
                '''
                echo 'STAGING E2E Completed'
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2E STAGING', reportTitles: '', useWrapperFileDirectly: true])
                }
            } 
        }

        stage('Approval') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    input message: 'Ready to deploy?', ok: 'Yes, I am sure I want to deploy'
                }
            }
        }

        stage('Deploy PRODUCTION') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
        
            steps {
                echo 'DEPLOYING TO PRODUCTION...'
                sh '''
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Delploying to test SITE ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }

        stage('Prod E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            
            environment {
                CI_ENVIRONMENT_URL = 'https://stately-klepon-30a3f7.netlify.app'
            }
            
            steps {
                echo 'RUN TEST...'
                sh '''
                    npx playwright test --reporter=html
                '''
                echo 'PROD E2E Completed'
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2E Prod', reportTitles: '', useWrapperFileDirectly: true])
                }
            } 
        }
    }   
}
