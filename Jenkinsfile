pipeline {
    agent any

    environment {
        // NETLIFY_SITE_ID = '5ad08bd9-02fe-4ef0-b55a-cb6630ea5b20'
        // NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        // AWS_S3_BUCKET = 'learn-jenkins-20241226-1505'
        REACT_APP_VERSION = "1.0.$BUILD_ID"
        AWS_DEFAULT_REGION = 'us-east-1'
        APP_NAME = 'learnjenkinsapp'
        AWS_ECR_REPO_ID = '701432201591.dkr.ecr.us-east-1.amazonaws.com'
        AWS_ECS_REV = ''
        AWS_ECS_CLUSTER = 'learnjenkins-cluster-prod'
        AWS_ECS_SERVICE = 'Jenkins-Service-Prod'
        AWS_ECS_TASKDEF_PATH = 'aws/jenkins-taskdef-prod.json'
        AWS_ECS_TASKDEF_PROD = 'Jenkins-TaskDefinition-Prod'

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

        stage('Build Docker Image') {

            agent {
                docker {
                    image 'my-aws'
                    args "-u root -v /var/run/docker.sock:/var/run/docker.sock --entrypoint=''"
                    reuseNode true
                }
            }

            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-s3', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        docker build -t $AWS_ECR_REPO_ID/$APP_NAME:$REACT_APP_VERSION .
                        aws ecr get-login-password | docker login --username AWS --password-stdin $AWS_ECR_REPO_ID
                        docker push $AWS_ECR_REPO_ID/$APP_NAME:$REACT_APP_VERSION
                    '''
                }
            }
        }


        stage('Deploy to AWS') {
            agent {
                docker {
                    image 'my-aws'
                    args "--entrypoint=''"
                    reuseNode true
                }
            }

            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-s3', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        aws --version
                        sed -i "s/#APP_VERSION#/$REACT_APP_VERSION/g" $AWS_ECS_TASKDEF_PATH
                        AWS_ECS_REV=$(aws ecs register-task-definition --cli-input-json file://$AWS_ECS_TASKDEF_PATH --query 'taskDefinition.revision' --output text)
                        aws ecs update-service --cluster $AWS_ECS_CLUSTER --service $AWS_ECS_SERVICE --task-definition $AWS_ECS_TASKDEF_PROD:${AWS_ECS_REV}
                        aws ecs wait services-stable --cluster $AWS_ECS_CLUSTER --services $AWS_ECS_SERVICE
                    '''

                }
            }
        }

        // stage('Run Tests Local') {
        //     parallel {
        //         stage('Test') {
                
        //             agent {
        //                 docker {
        //                     image 'node:18-alpine'
        //                     reuseNode true
        //                 }
        //             }

        //             steps {
        //                 echo 'RUN TEST...'
        //                 sh '''
        //                     test -f build/index.html
        //                     npm test
        //                 '''
        //             }

        //             post {
        //                 always {
        //                     junit 'jest-results/junit.xml'
        //                 }
        //             }
        //         }

        //         stage('E2E') {
                
        //             agent {
        //                 docker {
        //                     image 'my-playwright'
        //                     reuseNode true
        //                 }
        //             }

        //             steps {
        //                 echo 'RUN TEST...'
        //                 sh '''
        //                     serve -s build &
        //                     sleep 10
        //                     npx playwright test --reporter=html
        //                 '''
        //             }

        //             post {
        //                 always {
        //                     publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright', reportTitles: '', useWrapperFileDirectly: true])
        //                 }
        //             }
        //         }
        //     }
        // }

        // /*
        //  stage('Deploy STAGING') {
        //      agent {
        //          docker {
        //              image 'node:18-alpine'
        //              reuseNode true
        //          }
        //      }

        //      steps {
        //          echo 'DEPLOYING TO STAGING...'
        //          sh '''
        //              npm install netlify-cli node-jq
        //              node_modules/.bin/netlify --version
        //              echo "Delploying to test SITE ID: $NETLIFY_SITE_ID"
        //              node_modules/.bin/netlify status
        //              node_modules/.bin/netlify deploy --dir=build --json > staging.json
        //              node_modules/.bin/node-jq -r '.deploy_url' staging.json
        //          ''
        //          script {
        //              env.STAGE_URL = sh(script: "node_modules/.bin/node-jq -r '.deploy_url' staging.json", returnStdout: true)
        //          }
        //      }
        //  }
        // */

        // stage('DEPLOY STAGING') {
        //     agent {
        //         docker {
        //             image 'my-playwright'
        //             reuseNode true
        //         }
        //     }

        //     environment {
        //         CI_ENVIRONMENT_URL = 'STAGING_URL_TO_BE_SET'
        //     }

        //     steps {
        //         echo 'RUN TEST - STAGING...'
        //         sh '''
        //             netlify --version
        //             echo "Deploying to test SITE ID: $NETLIFY_SITE_ID"
        //             netlify status
        //             netlify deploy --dir=build --json > staging.json
        //             export CI_ENVIRONMENT_URL=$(node-jq -r '.deploy_url' staging.json)
        //             echo "CI_ENVIRONMENT_URL: $CI_ENVIRONMENT_URL"
        //             sleep 30
        //             npx playwright test --reporter=html
        //         '''
        //         echo 'STAGING E2E Completed'
        //           }

        //     post {
        //         always {
        //             publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2E STAGING', reportTitles: '', useWrapperFileDirectly: true])
        //         }
        //     } 
        // }


        // stage('Approval') {
        //     steps {
        //         timeout(time: 1, unit: 'MINUTES') {
        //             input message: 'Ready to deploy?', ok: 'Yes, I am sure I want to deploy'
        //         }
        //     }
        // }

        // /* 
        //  stage('Deploy PRODUCTION') {
        //      agent {
        //          docker {
        //              image 'node:18-alpine'
        //              reuseNode true
        //          }
        //      }

        //      steps {
        //          echo 'DEPLOYING TO PRODUCTION...'
        //          sh '''
        //              npm install netlify-cli
        //              node_modules/.bin/netlify --version
        //              echo "Delploying to test SITE ID: $NETLIFY_SITE_ID"
        //              node_modules/.bin/netlify status
        //              node_modules/.bin/netlify deploy --dir=build --prod
        //          '''
        //      }
        //  }
        // */

        // stage('DEPLOY PROD') {
        //     agent {
        //         docker {
        //             image 'my-playwright'
        //             reuseNode true
        //         }
        //     }
            
        //     environment {
        //         CI_ENVIRONMENT_URL = 'https://stately-klepon-30a3f7.netlify.app'
        //     }
            
        //     steps {
        //         echo 'RUN DEPLOY STAGE...'
        //         sh '''
        //             node --version
        //             npm --version
        //             netlify --version
        //             echo "Delploying to test SITE ID: $NETLIFY_SITE_ID"
        //             netlify status
        //             netlify deploy --dir=build --prod
        //             sleep 30
        //             npx playwright test --reporter=html
        //         '''
        //         echo 'DEPLOY PROD Completed'
        //     }

        //     post {
        //         always {
        //             publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2E Prod', reportTitles: '', useWrapperFileDirectly: true])
        //         }
        //     } 
        // }
    }   
}
