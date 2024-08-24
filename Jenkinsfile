pipeline {
    agent any

    environment {
        MONGO_DEB_URL = 'https://repo.mongodb.org/apt/ubuntu/dists/jammy/mongodb-org/7.0/multiverse/binary-amd64/mongodb-org-server_7.0.12_amd64.deb'
        REPO_URL = 'https://github.com/Ashutosh-Ahirwar/AquilaCMS.git'
        REPO_DIR = 'AquilaCMS'
        PORT = 3010  // Port on which the app should be listening
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
                    sh 'echo "y" | yarn install || true'  // Continue even if yarn install fails
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                dir(REPO_DIR) {
                    // Ensure all dependencies including dotenv are installed
                    sh 'yarn add dotenv'
                    sh 'yarn install || true'  // Continue even if yarn install fails
                }
            }
        }

        stage('Initial Start of Application') {
            steps {
                script {
                    dir(REPO_DIR) {
                        // Start the application and capture the output in a file
                        sh 'npm start > app.log 2>&1 & echo $! > .pid'
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
                        // Start the application and capture the output in a file
                        sh 'npm start > app.log 2>&1 & echo $! > .pid'
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
                                if (logContent.contains("listening on port ${PORT}") || logContent.contains("Error loading the theme")) {
                                    echo 'App is listening on port ${PORT} despite theme error'
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

        stage('Compile Themes and Final Start of Application') {
            steps {
                script {
                    dir(REPO_DIR) {
                        // Compile the themes
                        sh 'cd apps/themes/default_theme_2/ && npm run build'

                        // Final npm start (app should be fully operational)
                        sh 'npm start > app.log 2>&1 & echo $! > .pid'
                        
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
