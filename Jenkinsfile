pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'd3e939bc-bff4-4343-96b3-715de35c6758'
        // credentials -> va a prendere direttamente dal context di jenkins (config via dashboard)
        NETLIFY_AUTH_TOKEN = credentials('netlify_token_2')
    }

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    // non ricrea un nuovo workspace, mantiene i dati sincronizzati
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "started build phase"
                    ls -la
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Run Tests') {
            // avvia gli stage in maniera asincrona, attenzione a unire stage di lunghezze troppo diverse
            parallel {
                stage('Unit') {
                    agent {
                        docker {
                            image 'node:18-alpine'
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

                stage('E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                    npm i serve
                    node_modules/.bin/serve -s build &
                    sleep 10
                    npx playwright test --reporter=html
                '''
                    }

                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright LOCAL', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('StagingDeploy') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                  npm i netlify-cli@20.1.1 node-jq
                  node_modules/.bin/netlify status
                  node_modules/.bin/netlify deploy --dir=build --json > deploy.json
                  node_modules/.bin/node-jq -r './deploy_url' deploy.json
                '''
            }
        }
        stage('Approval') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Proceed with deployment?', ok: 'Yes, deploy!'
                }
            }
        }
        stage('Deploy') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                  npm i netlify-cli@20.1.1
                  node_modules/.bin/netlify status
                  node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }

        stage('Prod E2E Tests') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://aesthetic-begonia-156de6.netlify.app'
            }
            steps {
                sh '''
                    npx playwright test --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright REMOTE', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
    }}

    post {
        always {
            junit 'jest-results/junit.xml'
        }
    }
    }
