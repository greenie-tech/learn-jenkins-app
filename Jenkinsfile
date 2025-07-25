pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '56b5332e-fcf9-47db-a532-7b5159029fee'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
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

        stage('Test') {
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
                    test -f build/index.html
                    npm test
                '''
            }
        }

        stage('Deploy') {
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
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to Production. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }
    
    }

    post {
        always {
            junit 'test-results/junit.xml'
        }
    }
}
