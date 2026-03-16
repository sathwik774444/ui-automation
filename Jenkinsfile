pipeline {
    agent any

    environment {
        PYTHON_VERSION = '3.11'
        SELENIUM_HUB_URL = 'http://localhost:4444'
        WORKSPACE_DIR = "${WORKSPACE}"
        REPORTS_DIR = "${WORKSPACE}/reports"
        ALLURE_RESULTS_DIR = "${REPORTS_DIR}/allure-results"
        JUNIT_REPORT = "${REPORTS_DIR}/junit.xml"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 60, unit: 'MINUTES')
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    echo "Checked out branch: ${env.BRANCH_NAME}"
                    echo "Commit: ${env.GIT_COMMIT}"
                }
            }
        }

        stage('Setup Python Environment') {
            steps {
                script {
                    // Check if Python is available
                    bat 'python --version || python3 --version'
                    
                    // Create virtual environment
                    bat 'python -m venv venv || python3 -m venv venv'
                    
                    // Activate virtual environment and install dependencies
                    if (isUnix()) {
                        sh '''
                            source venv/bin/activate
                            pip install --upgrade pip
                            pip install -r requirements.txt
                        '''
                    } else {
                        bat '''
                            venv\\Scripts\\activate
                            python -m pip install --upgrade pip
                            pip install -r requirements.txt
                        '''
                    }
                }
            }
        }

        stage('Start Selenium Grid') {
            steps {
                script {
                    echo "Starting Selenium Grid..."
                    bat 'docker-compose up -d'
                    
                    // Wait for Selenium Hub to be ready
                    bat '''
                        echo "Waiting for Selenium Hub to be ready..."
                        powershell -Command "try { $timeout = 60; $start = Get-Date; do { Start-Sleep 2; try { Invoke-WebRequest -Uri http://localhost:4444/status -UseBasicParsing -TimeoutSec 5; Write-Host 'Selenium Hub is ready!'; exit 0 } catch { if ((Get-Date) -lt $start.AddSeconds($timeout)) { continue } else { exit 1 } } } while ($true) } catch { exit 1 }"
                    '''
                }
            }
        }

        stage('Create Reports Directory') {
            steps {
                bat 'if not exist "%REPORTS_DIR%" mkdir "%REPORTS_DIR%"'
                bat 'if not exist "%ALLURE_RESULTS_DIR%" mkdir "%ALLURE_RESULTS_DIR%"'
            }
        }

        stage('Run Tests') {
            parallel {
                stage('Smoke Tests') {
                    steps {
                        script {
                            if (isUnix()) {
                                sh '''
                                    source venv/bin/activate
                                    pytest -v -m smoke \
                                        --alluredir=${ALLURE_RESULTS_DIR} \
                                        --junitxml=${REPORTS_DIR}/junit-smoke.xml \
                                        --tb=short
                                '''
                            } else {
                                bat '''
                                    venv\\Scripts\\activate
                                    pytest -v -m smoke ^
                                        --alluredir=%ALLURE_RESULTS_DIR% ^
                                        --junitxml=%REPORTS_DIR%\\junit-smoke.xml ^
                                        --tb=short
                                '''
                            }
                        }
                    }
                    post {
                        always {
                            publishTestResults testResultsPattern: 'reports/junit-smoke.xml'
                            allure includeProperties: false, jdk: '', results: [[path: 'reports/allure-results']]
                        }
                    }
                }

                stage('Regression Tests') {
                    steps {
                        script {
                            if (isUnix()) {
                                sh '''
                                    source venv/bin/activate
                                    pytest -v -m regression \
                                        --alluredir=${ALLURE_RESULTS_DIR} \
                                        --junitxml=${REPORTS_DIR}/junit-regression.xml \
                                        --tb=short \
                                        -n 3
                                '''
                            } else {
                                bat '''
                                    venv\\Scripts\\activate
                                    pytest -v -m regression ^
                                        --alluredir=%ALLURE_RESULTS_DIR% ^
                                        --junitxml=%REPORTS_DIR%\\junit-regression.xml ^
                                        --tb=short ^
                                        -n 3
                                '''
                            }
                        }
                    }
                    post {
                        always {
                            publishTestResults testResultsPattern: 'reports/junit-regression.xml'
                            allure includeProperties: false, jdk: '', results: [[path: 'reports/allure-results']]
                        }
                    }
                }
            }
        }

        stage('Generate Allure Report') {
            steps {
                script {
                    if (isUnix()) {
                        sh '''
                            source venv/bin/activate
                            allure generate reports/allure-results -o reports/allure-report --clean
                        '''
                    } else {
                        bat '''
                            venv\\Scripts\\activate
                            allure generate reports\\allure-results -o reports\\allure-report --clean
                        '''
                    }
                }
            }
            post {
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'reports/allure-report',
                        reportFiles: 'index.html',
                        reportName: 'Allure Test Report'
                    ])
                }
            }
        }
    }

    post {
        always {
            script {
                echo 'Cleaning up...'
                // Archive test results and reports
                archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true
                
                // Archive logs if they exist
                archiveArtifacts artifacts: 'logs/**/*', allowEmptyArchive: true
                
                echo 'Pipeline completed!'
            }
        }

        success {
            script {
                echo '✅ All tests passed successfully!'
                // Send success notification (customize as needed)
                // emailext subject: "✅ UI Automation Success: ${env.JOB_NAME} - ${env.BUILD_NUMBER}", 
                //          body: "All tests passed. View report: ${env.BUILD_URL}Allure", 
                //          to: "${env.CHANGE_AUTHOR_EMAIL}"
            }
        }

        failure {
            script {
                echo '❌ Tests failed!'
                // Send failure notification (customize as needed)
                // emailext subject: "❌ UI Automation Failure: ${env.JOB_NAME} - ${env.BUILD_NUMBER}", 
                //          body: "Tests failed. View logs: ${env.BUILD_URL}console", 
                //          to: "${env.CHANGE_AUTHOR_EMAIL}"
            }
        }

        cleanup {
            script {
                echo 'Stopping Selenium Grid...'
                bat 'docker-compose down || true'
                
                // Clean up workspace (optional)
                cleanWs()
            }
        }
    }
}
