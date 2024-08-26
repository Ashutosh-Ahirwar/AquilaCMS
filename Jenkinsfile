pipeline {
    agent any

    options {
        // Abort any previous running builds when a new build starts
        disableConcurrentBuilds(abortPrevious: true)
    }

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

        stage('First Start of Application') {
            steps {
                script {
                    dir(REPO_DIR) {
                        // Start the application in detached mode using nohup
                        sh 'nohup npm start > app.log 2>&1 & echo $! > .pid'
                        sleep 10  // Wait a few seconds for the app to start

                        // Monitor the logs for the specific error continuously
                        def errorDetected = false
                        while (true) {
                            if (fileExists('app.log')) {
                                def logContent = readFile('app.log')
                                echo "Log Content: ${logContent}"
                                if (logContent.contains("Error: Command failed: yarn install")) {
                                    echo 'Error detected in logs: Command failed: yarn install'
                                    errorDetected = true
                                    break
                                }
                            }
                            echo 'Error not found yet, continuing to monitor...'
                            sleep 20  // Wait before checking again
                        }

                        if (errorDetected) {
                            // Stop the app process
                            sh 'kill $(cat .pid) || true'
                        } else {
                            error 'Expected error did not appear in the logs.'
                        }
                    }
                }
            }
        }

        stage('Second Start of Application') {
            steps {
                script {
                    dir(REPO_DIR) {
                        // Start the application in detached mode using nohup
                        sh 'nohup npm start > app.log 2>&1 & echo $! > .pid'
                        sleep 10  // Wait a few seconds for the app to start

                        // Monitor the logs for the theme error
                        def maxAttempts = 20  // Adjust this value if necessary
                        def attempt = 0
                        def errorDetected = false
                        while (attempt < maxAttempts) {
                            if (fileExists('app.log')) {
                                def logContent = readFile('app.log')
                                echo "Log Content: ${logContent}"
                                if (logContent.contains("Error loading the theme")) {
                                    echo 'Error detected in logs: Error loading the theme'
                                    errorDetected = true
                                    break
                                }
                            }
                            echo 'Error not found yet, retrying...'
                            sleep 20  // Wait before checking again
                            attempt++
                        }

                        if (errorDetected) {
                            // Stop the app process
                            sh 'kill $(cat .pid) || true'
                        } else {
                            error 'Expected error did not appear in the logs within the expected time.'
                        }
                    }
                }
            }
        }

        stage('Final Start of Application') {
            steps {
                script {
                    dir(REPO_DIR) {
                        // Compile the themes
                        sh 'cd apps/themes/default_theme_2/ && npm run build'

                        // Final npm start (app should be fully operational) using nohup
                        sh 'nohup npm start > app.log 2>&1 & echo $! > .pid'
                        
                        // Wait for app to be ready
                        def maxAttempts = 20  // Adjust this value if necessary
                        def attempt = 0
                        def success = false
                        while (attempt < maxAttempts) {
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
                            sleep 20
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
            // No need to kill process here as it's already managed
        }
    }
}
