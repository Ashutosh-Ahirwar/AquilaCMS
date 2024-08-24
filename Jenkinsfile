pipeline {
    agent any

    environment {
        MONGO_DEB_URL = 'https://repo.mongodb.org/apt/ubuntu/dists/jammy/mongodb-org/7.0/multiverse/binary-amd64/mongodb-org-server_7.0.12_amd64.deb'
        REPO_URL = 'https://github.com/Ashutosh-Ahirwar/AquilaCMS.git'
        REPO_DIR = 'AquilaCMS'
        PORT = '3010'
    }

    stages {
        stage('Install MongoDB') {
            steps {
                sh 'curl ${MONGO_DEB_URL} -O'
                sh 'sudo dpkg -i mongodb-org-server_7.0.12_amd64.deb'
                sh 'sudo systemctl start mongod'
            }
        }

        stage('Install Node.js and Corepack') {
            steps {
                sh 'sudo snap install node --classic'
                sh 'npm uninstall -g yarn pnpm -y'
                sh 'sudo npm install -g corepack -y'
            }
        }

        stage('Clone or Pull Repository') {
            steps {
                script {
                    if (fileExists(REPO_DIR)) {
                        dir(REPO_DIR) {
                            sh 'git pull'
                        }
                    } else {
                        sh 'git clone ${REPO_URL}'
                    }
                }
            }
        }

        stage('Enable Corepack and Install Yarn') {
            steps {
                dir(REPO_DIR) {
                    sh 'echo "y" | corepack enable'
                    sh 'yarn set version stable'
                    
                    // Install dependencies with error handling
                    script {
                        def installStatus = sh(script: 'yarn install', returnStatus: true)
                        if (installStatus != 0) {
                            echo 'yarn install completed with errors, but continuing pipeline'
                        }
                    }
                }
            }
        }

        stage('Initial Start of Application') {
            steps {
                script {
                    dir(REPO_DIR) {
                        // Start the application and redirect logs to a file
                        sh 'npm start &> app.log & echo $! > .pid'
                        sleep 10  // Wait a few seconds for the app to start

                        // Check if the app is listening on the port or if it encountered an error
                        def maxAttempts = 10
                        def attempt = 0
                        def success = false
                        while (attempt < maxAttempts) {
                            sh 'sudo ss -tuln'  // Print all listening ports
                            if (fileExists('app.log')) {
                                def logContent = readFile('app.log')
                                echo "Log Content: ${logContent}"
                                if (logContent.contains("listening on port ${PORT}")) {
                                    echo 'App is listening on port ${PORT}. Waiting for potential errors...'
                                    success = true
                                    break
                                } else if (logContent.contains("Error: Command failed: yarn install")) {
                                    echo 'Error occurred: yarn install failed'
                                    success = true
                                    break
                                }
                            }
                            echo 'App not ready yet or error not found, retrying...'
                            sleep 10
                            attempt++
                        }

                        if (!success) {
                            error 'App did not start correctly or did not find the specific error within the expected time.'
                        }

                        // Stop the app process after confirmation
                        sh 'kill $(cat .pid) || true'
                    }
                }
            }
        }

        stage('Second Start of Application') {
            steps {
                script {
                    dir(REPO_DIR) {
                        // Start the application again and redirect logs to a file
                        sh 'npm start &> app.log & echo $! > .pid'
                        sleep 10  // Wait a few seconds for the app to start

                        // Check if the app is listening on the port or if it encountered a theme loading error
                        def maxAttempts = 10
                        def attempt = 0
                        def success = false
                        while (attempt < maxAttempts) {
                            sh 'sudo ss -tuln'  // Print all listening ports
                            if (fileExists('app.log')) {
                                def logContent = readFile('app.log')
                                echo "Log Content: ${logContent}"
                                if (logContent.contains("listening on port ${PORT}")) {
                                    echo 'App is listening on port ${PORT}. Waiting for theme error...'
                                    success = true
                                    break
                                } else if (logContent.contains("Error loading the theme")) {
                                    echo 'Error occurred: Loading the theme failed'
                                    success = true
                                    break
                                }
                            }
                            echo 'App not ready yet or theme error not found, retrying...'
                            sleep 10
                            attempt++
                        }

                        if (!success) {
                            error 'App did not start correctly or did not find the theme loading error within the expected time.'
                        }

                        // Stop the app process after confirmation
                        sh 'kill $(cat .pid) || true'
                    }
                }
            }
        }

        stage('Compile Themes and Final Start of Application') {
            steps {
                script {
                    dir(REPO_DIR) {
                        // Compile the themes
                        sh 'cd apps/themes/default_theme_2/ && npm run build'

                        // Final npm start (app should be fully operational)
                        sh 'npm start &> app.log & echo $! > .pid'
                        
                        // Wait for app to be ready
                        def maxAttempts = 10
                        def attempt = 0
                        def success = false
                        while (attempt < maxAttempts) {
                            sh 'sudo ss -tuln'  // Print all listening ports
                            if (fileExists('app.log')) {
                                def logContent = readFile('app.log')
                                echo "Log Content: ${logContent}"
                                if (logContent.contains("listening on port ${PORT}")) {
                                    echo 'App is successfully listening on port ${PORT}'
                                    success = true
                                    break
                                }
                            }
                            echo 'App not ready yet, retrying...'
                            sleep 10
                            attempt++
                        }

                        if (!success) {
                            error 'App did not start correctly within the expected time.'
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '**/AquilaCMS/**/*', allowEmptyArchive: true
            sh 'kill $(cat .pid) || true' // Ensure the app is stopped
        }
    }
}
