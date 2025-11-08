pipeline {
    agent none   // We’ll pick the right agent per-stage

    environment {
        // Makes npm cache survive between builds → 10× faster
        NPM_CONFIG_CACHE = "${workspace}/.npm"
    }

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    label 'docker'          // any node with Docker
                    reuseNode true
                }
            }
            steps {
                sh '''
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la build
                '''
                stash name: 'built', includes: 'build/**, package.json, package-lock.json'
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            label 'docker'
                            reuseNode true
                        }
                    }
                    steps {
                        unstash 'built'
                        sh 'npm test'
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
                            label 'docker'
                            reuseNode true
                            args '-u root'   // needed for apt-get inside container
                        }
                    }
                    steps {
                        unstash 'built'
                        sh '''
                            npm install -g serve
                            serve -s build &
                            SERVE_PID=$!
                            sleep 10
                            npx playwright test --reporter=html
                            kill $SERVE_PID || true
                        '''
                    }
                    post {
                        always {
                            publishHTML(
                                target: [
                                    reportDir: 'playwright-report',
                                    reportFiles: 'index.html',
                                    reportName: 'Playwright E2E Report',
                                    keepAll: true
                                ]
                            )
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            agent {
                docker {
                    image 'node:18-alpine'
                    label 'docker'
                    reuseNode true
                }
            }
            environment {
                NETLIFY_AUTH_TOKEN = credentials('netlify-token')
                NETLIFY_SITE_ID    = credentials('netlify-site-id')
            }
            steps {
                unstash 'built'
                sh '''
                    npm install -g netlify-cli
                    netlify --version
                    netlify deploy --dir=build --prod --site $NETLIFY_SITE_ID
                '''
            }
        }
    }
}