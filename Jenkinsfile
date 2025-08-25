pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '56b5332e-fcf9-47db-a532-7b5159029fee'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
        AWS_DEFAULT_REGION = 'us-east-1'

    }
    stages {
        
        stage('Docker') {
            steps {
                // build the docker image using the local Docker file. 
                // The -t is for tagging the image
                // The "." specifies the build context (current directory) where the Dockerfile is located
                sh 'docker build -t my-playwright-image .'
            }
        }

        stage('Build') {
            agent {
                docker{
                    //image 'node:18-alpine'
                    //image 'node:18-bullseye'
                    image 'node:20-bullseye'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    whoami
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }
        
        stage('AWS S3 Upload') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    reuseNode true
                    args "--entrypoint=''" //needed to prevent container from exiting immediately
                }
            }
            environment {
                AWS_S3_BUCKET = 'gtech-learn-jenkins'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'gtech-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        aws --version
                        #echo "Hello S3!" > index.html
                        pwd
                        ls
                        aws s3 ls
                        #aws s3 cp index.html s3://gtech-learn-jenkins/index.html
                        aws s3 sync build s3://$AWS_S3_BUCKET
                    '''
                }
            }
        }

         stage('AWS ECS Deploy') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    reuseNode true
                    args "--entrypoint=''" //needed to prevent container from exiting immediately
                }
            }
            /*
            environment {
                AWS_S3_BUCKET = 'gtech-learn-jenkins'
            }
            */
            steps {
                withCredentials([usernamePassword(credentialsId: 'gtech-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        aws --version
                        aws ecs register-task-definition --cli-input-json file://aws/task-definition-prod.json
                        aws ecs update-service --cluster LearnJenkinsApp-Cluster-Prod --service LearnJenkinsApp-Service-Prod --task-definition LearnJenkinsApp-TaskDefinition-Prod:2
                    '''
                }
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit Test') {
                    agent {
                        docker{
                            //image 'node:18-alpine'
                            //image 'node:18-bullseye'
                            image 'node:20-bullseye'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            echo "Test Stage"
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
                            //image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            image 'my-playwright-image'   // Using the custom image built earlier
                            reuseNode true
                            //args '-u root:root'  //why to run this as root *not recomended*
                        }
                    }

                    steps {
                        sh '''
                            ## logic for local e2e tests
                            #npm install serve
                            #node_modules/.bin/serve -s build &
                            ## global serve when using playwright image
                            serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'playwright-report Local E2E', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }

        }
        
        stage('Deploy Staging') {
            agent {
                docker{
                    //image 'node:18-alpine'
                    //image 'node:18-bullseye'
                    image 'node:20-bullseye'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    # npm install netlify-cli
                    npm install netlify-cli@23.0.0
                    npm install node-jq
                    node_modules/.bin/netlify --version
                    echo "Deploying to Staging. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                    node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json
                '''
                script {
                    env.STAGING_URL = sh(script: "node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json", returnStdout: true)
                }
            }
        }
        
        stage('Staging E2E') {
            agent {
                docker {
                    //image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    image 'my-playwright-image'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = "${env.STAGING_URL}"
            }

            steps {
                sh '''
                    npx playwright test  --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'playwright-report Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
        
        stage('Approval'){
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    input message: 'Ready to deploy?', ok: 'Yes, Let\'s go!'
                }
            }
        }
        stage('Deploy Prod') {
            agent {
                docker{
                    //image 'node:18-alpine'
                    //image 'node:18-bullseye'
                    image 'node:20-bullseye'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    # npm install netlify-cli
                    npm install netlify-cli@23.0.0
                    node_modules/.bin/netlify --version
                    echo "Deploying to Production. Site ID: $NETLIFY_SITE_ID"
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
                            //args '-u root:root'  //why to run this as root *not recomended*
                        }
                    }

                    environment {
                        CI_ENVIRONMENT_URL = 'https://lively-kulfi-5137a3.netlify.app'
                    }

                    steps {
                        sh '''
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'playwright-report Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
        } 
    }
}