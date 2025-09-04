pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '56b5332e-fcf9-47db-a532-7b5159029fee'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        APP_NAME = 'LearnJenkinsApp'
        REACT_APP_VERSION = "1.0.$BUILD_ID"
        AWS_DEFAULT_REGION = 'us-east-1'
        AWS_ECS_CLUSTER = 'LearnJenkinsApp-Cluster-Prod'
        AWS_ECS_SERVICE_PROD = 'LearnJenkinsApp-Service-Prod'
        AWS_ECS_TD_PROD = 'LearnJenkinsApp-TaskDefinition-Prod'

    }
    stages {
        /*
        stage('Build Docker Image (Playwright)') {
            steps {
                // build the docker image using the local Docker file in 'ci' directory.
                // The -t is for tagging the image
                // The "." specifies the build context (current directory) where the Dockerfile is located
                sh 'docker build -f ci/Dockerfile -t my-playwright-image .'
            }
        }
        */
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

                    # Install dependencies (clean install)
                    npm ci

                    # Build the application
                    npm run build

                    # List the contents of directory after build
                    ls -la
                '''
            }
        }
        
        stage('Build Docker Image (myjenkinsapp)') {
            agent {
                docker {
                    /* Use the Amazon AWS CLI image
                    image 'amazon/aws-cli'
                    */

                    // Use custom image with docker installed
                    image 'my-aws-cli'

                    // Reuse the same Docker node
                    reuseNode true

                    // -u root execute as root user
                    // Mount the Docker socket using -v /var/run/docker.sock:/var/run/docker.sock
                    // --entrypoint='' needed to prevent container from exiting immediately
                    args "-u root -v /var/run/docker.sock:/var/run/docker.sock --entrypoint=''"
                }
            }
            steps {  
                sh '''
                    # Print the image ID to get version details
                    cat /etc/image-id

                    # Install Docker for amazon linux
                        # This amazon-linux-extras command doesn't work on 2023 aws cli image
                        # amazon-linux-extras install docker

                        # Install Docker using dnf on Amazon Linux 2023
                        dnf install docker -y

                    # build the docker image using the local Docker file
                    # The -t is for tagging the image
                    # The "." specifies the build context (current directory) where the Dockerfile is located
                    docker build -t $APP_NAME:$REACT_APP_VERSION .
                '''
            }
        }

        stage('AWS S3 Upload') {
            agent {
                docker {
                    // Use the Amazon AWS CLI image
                    //image 'amazon/aws-cli'

                    // Using the custom image from nightly builds
                    image 'my-aws-cli'

                    // Reuse the same Docker node
                    reuseNode true

                    // Prevent container from exiting immediately
                    args "--entrypoint=''" //needed to prevent container from exiting immediately
                }
            }
            environment {
                AWS_S3_BUCKET = 'gtech-learn-jenkins'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'gtech-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        # Print the AWS CLI version
                        aws --version
                        #echo "Hello S3!" > index.html

                        # Print the current working directory
                        pwd

                        # List files in the current directory
                        ls

                        # List S3 bucket contents
                        aws s3 ls
                        #aws s3 cp index.html s3://gtech-learn-jenkins/index.html

                        # Sync the build directory to S3
                        aws s3 sync build s3://$AWS_S3_BUCKET
                    '''
                }
            }
        }

         stage('AWS ECS Deploy') {
            agent {
                docker {
                    // Use the Amazon AWS CLI image
                      //image 'amazon/aws-cli'
                    image 'my-aws-cli'

                    // Reuse the same Docker node
                    reuseNode true

                    // Prevent container from exiting immediately
                    args "-u root --entrypoint=''" //needed to execute as root user and prevent container from exiting immediately
                }
            }
            /*
            environment {
                AWS_S3_BUCKET = 'gtech-learn-jenkins'
            }
            */
            steps {
                // Deploy to AWS ECS
                withCredentials([usernamePassword(credentialsId: 'gtech-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        # Install AWS CLI v2
                        aws --version

                        # Install jq for JSON parsing
                        yum install jq -y

                        # Register new task definition from local json file and capture the revision number using jq
                        LATEST_TD_REVISION=$(aws ecs register-task-definition --cli-input-json file://aws/task-definition-prod.json | jq '.taskDefinition.revision')

                        # Update the existing ECS service to use the new task definition revision
                        aws ecs update-service --cluster $AWS_ECS_CLUSTER --service $AWS_ECS_SERVICE_PROD --task-definition $AWS_ECS_TD_PROD:$LATEST_TD_REVISION

                        # Wait for the ECS service to stabilize
                        #aws ecs wait services-stable --cluster $AWS_ECS_CLUSTER --service $AWS_ECS_SERVICE_PROD
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
                            # Unit Test Stage
                            echo "Test Stage"

                            # Check if the build artifact exists
                            test -f build/index.html

                            # Run the tests
                            npm test
                        '''
                    }
                    post {
                        always {
                            // Publish JUnit test results
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            //image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            //image 'my-playwright-image'   // Using the custom image built earlier
                            image 'my-playwright' // Using the custom image built from nightly build

                            // Reuse the same Docker node
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
                    //image 'my-playwright-image'
                    image 'my-playwright'
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