pipeline {
    agent any

    environment {
        MONGO_DEB_URL = 'https://repo.mongodb.org/apt/ubuntu/dists/jammy/mongodb-org/7.0/multiverse/binary-amd64/mongodb-org-server_7.0.12_amd64.deb'
        REPO_URL = 'https://github.com/Ashutosh-Ahirwar/AquilaCMS.git'
        REPO_DIR = 'AquilaCMS'
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

        stage('Start Application and Wait for Logs') {
            steps {
                script {
                    dir(REPO_DIR) {
                        sh 'npm start &> app.log & echo $! > .pid' // Start the app and save the process ID
                        sleep 10  // Wait a few seconds for the app to start
                        
                        // Wait for a specific error message or success indication in the log
                        def maxAttempts = 20
                        def attempt = 0
                        def success = false
                        while (attempt < maxAttempts) {
                            if (fileExists('app.log')) {
                                def logContent = readFile('app.log')
                                if (logContent.contains("Error loading the theme") || logContent.contains("listening on port 3010")) {
                                    echo 'App has started and logs contain expected content'
                                    success = true
                                    break
                                }
                            }
                            echo 'App not ready yet, retrying...'
                            sleep 15  // Wait before checking again
                            attempt++
                        }

                        if (!success) {
                            error 'App did not start correctly within the expected time.'
                        }
                    }
                }
            }
        }

        stage('Compile Themes and Restart Application') {
            steps {
                script {
                    dir(REPO_DIR) {
                        sh 'kill $(cat .pid)' // Stop the first run of the application

                        // Compile themes
                        sh 'cd apps/themes/default_theme_2/ && npm run build'

                        // Start the app again
                        sh 'npm start &> app.log & echo $! > .pid'
                        
                        // Wait for app to be ready again
                        def maxAttempts = 20
                        def attempt = 0
                        def success = false
                        while (attempt < maxAttempts) {
                            if (fileExists('app.log')) {
                                def logContent = readFile('app.log')
                                if (logContent.contains("listening on port 3010")) {
                                    echo 'App restarted and is now listening on port 3010'
                                    success = true
                                    break
                                }
                            }
                            echo 'App not ready yet, retrying...'
                            sleep 15  // Wait before checking again
                            attempt++
                        }

                        if (!success) {
                            error 'App did not restart correctly within the expected time.'
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
