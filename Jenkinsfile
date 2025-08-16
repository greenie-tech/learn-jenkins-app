pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '56b5332e-fcf9-47db-a532-7b5159029fee'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = '1.2.3'
    }
    stages {
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
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                            //args '-u root:root'  //why to run this as root *not recomended*
                        }
                    }

                    steps {
                        sh '''
                            npm install serve
                            #serve -s build
                            node_modules/.bin/serve -s build &
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
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
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
                timeout(time: 1, unit: 'MINUTES') {
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