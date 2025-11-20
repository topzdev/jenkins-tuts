pipeline {
    agent any
    environment {
        NETLIFY_SITE_ID = 'd366b446-07ad-4f9c-aaa4-d82c455aad32'
        NETLIFY_AUTH_TOKEN = credentials('netlify-secret')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }

    stages {
        stage('Docker') {
            steps {
                sh 'Docker build -t deploy-image .'
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'node:20-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "Building..."
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
                stage('Test') {
                    agent {
                        docker {
                            image 'node:20-alpine'
                            reuseNode true
                        }
                    }

                    steps {
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
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy Staging') {
            agent {
                docker {
                    image 'deploy-build'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL="_" 
            }

            steps {
                echo "STAGING URL: ${env.STAGING_URL}"  
                sh '''
                    netlify --version
                    echo "Netlify Staging ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --json > staging-log.json
                    CI_ENVIRONMENT_URL=$(node-jq -r ".deploy_url" staging-log.json)
                    npx playwright test --reporter=html
                '''
            }
            
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        stage('Deploy Prod') {
            agent {
                docker {
                    image 'deploy-image'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL="https://voluble-strudel-d01c98.netlify.app" 
            }

            steps {
                sh '''
                    netlify --version
                    echo "Netlify Production ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --prod
                    npx playwright test --reporter=html
                '''
            }
            
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Live', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

    }

}