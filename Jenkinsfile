pipeline {
    agent any

    environment {
        MONGO_DEB_URL = 'https://repo.mongodb.org/apt/ubuntu/dists/jammy/mongodb-org/7.0/multiverse/binary-amd64/mongodb-org-server_7.0.12_amd64.deb'
        REPO_URL = 'https://github.com/Ashutosh-Ahirwar/AquilaCMS.git'
        REPO_DIR = 'AquilaCMS'
        PORT = 3010
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

        stage('First Start of Application') {
            steps {
                script {
                    dir(REPO_DIR) {
                        // Start the application
                        sh 'npm start &> app.log & echo $! > .pid'
                        sleep 10  // Wait a few seconds for the app to start

                        // Monitor the logs for specific errors
                        def maxAttempts = 10
                        def attempt = 0
                        def success = false
                        while (attempt < maxAttempts) {
                            if (fileExists('app.log')) {
                                def logContent = readFile('app.log')
                                if (logContent.contains("Error: Command failed: yarn install")) {
                                    echo 'Error detected: Command failed: yarn install. Terminating app.'
                                    sh 'kill $(cat .pid)'
                                    success = true
                                    break
                                }
                            }
                            echo 'App not ready yet, retrying...'
                            sleep 10
                            attempt++
                        }

                        if (!success) {
                            error 'Failed to detect the expected error in logs.'
                        }
                    }
                }
            }
        }

        stage('Second Start of Application') {
            steps {
                script {
                    dir(REPO_DIR) {
                        // Start the application
                        sh 'npm start &> app.log & echo $! > .pid'
                        sleep 10  // Wait a few seconds for the app to start

                        // Monitor the logs for specific errors
                        def maxAttempts = 10
                        def attempt = 0
                        def success = false
                        while (attempt < maxAttempts) {
                            if (fileExists('app.log')) {
                                def logContent = readFile('app.log')
                                if (logContent.contains("Theme start fail : Error: Error loading the theme")) {
                                    echo 'Error detected: Error loading the theme. Terminating app.'
                                    sh 'kill $(cat .pid)'
                                    success = true
                                    break
                                }
                            }
                            echo 'App not ready yet, retrying...'
                            sleep 10
                            attempt++
                        }

                        if (!success) {
                            error 'Failed to detect the expected error in logs.'
                        }
                    }
                }
            }
        }

        stage('Third Start of Application') {
            steps {
                script {
                    dir(REPO_DIR) {
                        // Start the application
                        sh 'npm start &> app.log & echo $! > .pid'
                        sleep 10  // Wait a few seconds for the app to start

                        // Monitor the logs for successful startup
                        def maxAttempts = 10
                        def attempt = 0
                        def success = false
                        while (attempt < maxAttempts) {
                            if (fileExists('app.log')) {
                                def logContent = readFile('app.log')
                                if (logContent.contains("server listening on port ${PORT}")) {
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
