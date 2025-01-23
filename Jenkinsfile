pipeline {
    agent any
    
    options {
        timeout(time: 10, unit: 'MINUTES')
    }
    
    environment {
        CURL_TIMEOUT = '30'
        RETRY_INTERVAL = '5'
    }
    
    parameters {
        string(
            name: 'SITE_NAME',
            defaultValue: 'poc-netlify-e2e-test',
            description: 'Netlify site name (e.g., yoursite.netlify.app)',
            trim: true
        )
        string(
            name: 'BRANCH_NAME',
            defaultValue: BRANCH_NAME,
            description: 'Branch name or PR number (e.g., PR-1234, main, develop)',
            trim: true
        )
        booleanParam(
            name: 'IS_PR',
            defaultValue: BRANCH_NAME ? BRANCH_NAME.startsWith('PR-') : false,
            description: 'Whether this is a PR deployment (auto-detected from BRANCH_NAME)'
        )
        string(
            name: 'MAX_TIMEOUT',
            defaultValue: '300',
            description: 'Maximum time (in seconds) to wait for the Netlify deploy preview URL',
            trim: true
        )
        string(
            name: 'BASE_PATH',
            defaultValue: '/',
            description: 'Base path of the preview URL to check (e.g., / or /your-page)',
            trim: true
        )
        booleanParam(
            name: 'FAIL_ON_ERROR',
            defaultValue: true,
            description: 'Whether to fail the build if preview URL is not accessible'
        )
    }

    stages {
        stage('Validate Parameters') {
            steps {
                script {
                    // Validate required parameters
                    if (!params.SITE_NAME?.trim()) {
                        error 'SITE_NAME parameter is required.'
                    }
                    if (!params.BRANCH_NAME?.trim()) {
                        error 'BRANCH_NAME parameter is required.'
                    }
                    
                    // Validate timeout value
                    try {
                        if (params.MAX_TIMEOUT.toInteger() < 60) {
                            error 'MAX_TIMEOUT must be at least 60 seconds.'
                        }
                    } catch (NumberFormatException e) {
                        error 'MAX_TIMEOUT must be a valid integer.'
                    }
                    
                    // Validate BASE_PATH format
                    if (!params.BASE_PATH.startsWith('/')) {
                        error 'BASE_PATH must start with a forward slash (/)'
                    }

                    // Log deployment type
                    echo "Deployment type: ${params.IS_PR ? 'Pull Request' : 'Branch'}"
                    echo "Branch name: ${params.BRANCH_NAME}"
                }
            }
        }

        stage('Construct Preview URL') {
            steps {
                script {
                    try {
                        if (params.IS_PR) {
                            // Handle PR deployments
                            def prMatcher = params.BRANCH_NAME =~ /PR-(\d+)/
                            if (!prMatcher.matches()) {
                                error "Branch name '${params.BRANCH_NAME}' is marked as PR but doesn't match expected format 'PR-number'"
                            }
                            env.PULL_REQUEST_NUMBER = prMatcher[0][1]
                            env.PREVIEW_URL = "https://deploy-preview-${env.PULL_REQUEST_NUMBER}--${params.SITE_NAME}.netlify.app${params.BASE_PATH}"
                            echo "PR number detected: ${env.PULL_REQUEST_NUMBER}"
                        } else {
                            // Handle branch deployments
                            def netlifyBranchName = params.BRANCH_NAME
                                .toLowerCase()
                                .replaceAll('/', '-')
                                .replaceAll('-+', '-')
                            env.PREVIEW_URL = "https://${netlifyBranchName}--${params.SITE_NAME}.netlify.app${params.BASE_PATH}"
                            echo "Branch name formatted for Netlify: ${netlifyBranchName}"
                        }
                        
                        echo "🔗 Constructed Netlify Preview URL: ${env.PREVIEW_URL}"
                    } catch (Exception e) {
                        error "Failed to construct preview URL: ${e.getMessage()}"
                    }
                }
            }
        }

        stage('Wait for Preview Deployment') {
            steps {
                script {
                    def urlReady = false
                    def maxTimeout = params.MAX_TIMEOUT.toInteger()
                    def retries = maxTimeout / RETRY_INTERVAL.toInteger()
                    def startTime = System.currentTimeMillis()
                    
                    for (int i = 0; i < retries; i++) {
                        try {
                            def curlCommand = """
                                curl -o /dev/null 
                                    -s 
                                    -w '%{http_code}' 
                                    --max-time ${CURL_TIMEOUT}
                                    --retry 3
                                    --retry-delay 1
                                    '${env.PREVIEW_URL}'
                            """.stripIndent().replaceAll('\n', ' ')
                            
                            def responseCode = sh(script: curlCommand, returnStdout: true).trim()
                            
                            if (responseCode == '200') {
                                echo "✅ Netlify Preview URL is ready: ${env.PREVIEW_URL}"
                                urlReady = true
                                break
                            } else {
                                def elapsedSeconds = (System.currentTimeMillis() - startTime) / 1000
                                echo "⏳ Netlify URL not ready (Code: ${responseCode}). Elapsed time: ${elapsedSeconds}s"
                            }
                        } catch (Exception e) {
                            echo "⚠️ Error checking URL: ${e.getMessage()}"
                        }
                        
                        sleep time: RETRY_INTERVAL.toInteger(), unit: 'SECONDS'
                    }

                    if (!urlReady && params.FAIL_ON_ERROR) {
                        error "❌ Timeout reached: Unable to connect to ${env.PREVIEW_URL}"
                    } else if (!urlReady) {
                        echo "⚠️ Warning: Preview URL not available, but continuing due to FAIL_ON_ERROR=false"
                    }
                }
            }
        }

        stage('Run Tests') {
            when {
                expression { env.PREVIEW_URL != null }
            }
            parallel {
                stage('Lighthouse Test') {
                    steps {
                        echo "🔍 TODO: Add Lighthouse performance testing"
                        // Add lighthouse testing steps here
                    }
                }
                stage('E2E Tests') {
                    steps {
                        echo "🧪 TODO: Add E2E testing"
                        // Add Cypress or other E2E testing steps here
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully! Preview URL: ${env.PREVIEW_URL}"
        }
        failure {
            echo "❌ Pipeline failed! Check logs for details."
        }
        always {
            cleanWs()
        }
    }
}
