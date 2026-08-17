pipeline {

    agent any

    parameters {
        string(
            name: 'CI_BUILD_NUMBER',
            defaultValue: '',
            description: 'Successful build number of Java_CI_Project'
        )
    }

    environment {

        // ==============================
        // CI CONFIGURATION
        // ==============================
        CI_JOB = 'Java_CI_Project'

        ARTIFACT = 'target/java-vm-demo-1.0.jar'


        // ==============================
        // EC2 CONFIGURATION
        // ==============================
        VM_HOST = '43.204.217.49'
        VM_USER = 'ec2-user'

        SSH_CREDENTIAL = 'ec2-ssh-key'


        // ==============================
        // APPLICATION CONFIGURATION
        // ==============================
        APP_NAME = 'java-vm-demo'
        APP_DIR = '/opt/java-app'
        JAR_NAME = 'java-vm-demo.jar'

        APP_PORT = '8080'
    }


    stages {

        // ==========================================================
        // 1. DOWNLOAD ARTIFACT FROM CI
        // ==========================================================

        stage('Download Artifact From CI') {

            steps {

                script {

                    if (!params.CI_BUILD_NUMBER?.trim()) {

                        error(
                            'CI_BUILD_NUMBER is required. ' +
                            'Enter the successful Java_CI_Project build number.'
                        )
                    }

                    echo """
                    ========================================
                    DOWNLOADING ARTIFACT
                    ========================================

                    CI Job     : ${CI_JOB}
                    CI Build   : ${params.CI_BUILD_NUMBER}
                    Artifact   : ${ARTIFACT}

                    ========================================
                    """

                    copyArtifacts(
                        projectName: CI_JOB,
                        selector: specific(params.CI_BUILD_NUMBER),
                        filter: ARTIFACT,
                        fingerprintArtifacts: true
                    )
                }
            }
        }


        // ==========================================================
        // 2. VERIFY ARTIFACT
        // ==========================================================

        stage('Verify Artifact') {

            steps {

                sh '''
                    set -e

                    echo "Checking downloaded artifact..."

                    ls -lh target/ || true

                    if [ ! -f target/java-vm-demo-1.0.jar ]; then
                        echo "ERROR: JAR file not found!"
                        exit 1
                    fi

                    echo "Artifact found successfully."

                    ls -lh target/java-vm-demo-1.0.jar

                    echo "Artifact verification successful."
                '''
            }
        }


        // ==========================================================
        // 3. TEST SSH CONNECTION
        // ==========================================================

        stage('Test SSH Connection') {

            steps {

                sshagent(credentials: [SSH_CREDENTIAL]) {

                    sh '''
                        set -e

                        echo "Testing SSH connection to EC2..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            -o ConnectTimeout=10 \
                            ${VM_USER}@${VM_HOST} \
                            "echo SSH connection successful; hostname; whoami"
                    '''
                }
            }
        }


        // ==========================================================
        // 4. INSTALL JAVA 17
        // ==========================================================

        stage('Install Java 17') {

            steps {

                sshagent(credentials: [SSH_CREDENTIAL]) {

                    sh '''
                        set -e

                        echo "Checking Java on EC2..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} '

                            if command -v java >/dev/null 2>&1
                            then

                                echo "Java already installed."
                                java -version

                            else

                                echo "Java not found."
                                echo "Installing Java 17..."

                                sudo dnf install -y java-17-amazon-corretto

                                echo "Java installation completed."

                                java -version

                            fi

                            '
                    '''
                }
            }
        }


        // ==========================================================
        // 5. PREPARE APPLICATION DIRECTORY
        // ==========================================================

        stage('Prepare Application') {

            steps {

                sshagent(credentials: [SSH_CREDENTIAL]) {

                    sh '''
                        set -e

                        echo "Preparing application directory..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} '

                            sudo mkdir -p ${APP_DIR}

                            sudo chown -R ${VM_USER}:${VM_USER} ${APP_DIR}

                            echo "Application directory prepared."

                            ls -ld ${APP_DIR}

                            '
                    '''
                }
            }
        }


        // ==========================================================
        // 6. COPY JAR TO EC2
        // ==========================================================

        stage('Copy JAR') {

            steps {

                sshagent(credentials: [SSH_CREDENTIAL]) {

                    sh '''
                        set -e

                        echo "Copying JAR to EC2..."

                        scp \
                            -o StrictHostKeyChecking=no \
                            target/java-vm-demo-1.0.jar \
                            ${VM_USER}@${VM_HOST}:${APP_DIR}/${JAR_NAME}

                        echo "JAR copied successfully."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} \
                            "ls -lh ${APP_DIR}/${JAR_NAME}"
                    '''
                }
            }
        }


        // ==========================================================
        // 7. CREATE SYSTEMD SERVICE
        // ==========================================================

        stage('Configure Systemd') {

            steps {

                sshagent(credentials: [SSH_CREDENTIAL]) {

                    sh '''
                        set -e

                        echo "Creating systemd service..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} '

                            sudo tee /etc/systemd/system/${APP_NAME}.service > /dev/null <<EOF

[Unit]
Description=Java VM Demo Spring Boot Application
After=network.target

[Service]
Type=simple
User=${VM_USER}
WorkingDirectory=${APP_DIR}
ExecStart=/usr/bin/java -jar ${APP_DIR}/${JAR_NAME}
Restart=always
RestartSec=5
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target

EOF

                            echo "Reloading systemd..."

                            sudo systemctl daemon-reload

                            echo "Enabling application service..."

                            sudo systemctl enable ${APP_NAME}

                            echo "Systemd configuration completed."

                            '
                    '''
                }
            }
        }


        // ==========================================================
        // 8. RESTART APPLICATION
        // ==========================================================

        stage('Deploy Application') {

            steps {

                sshagent(credentials: [SSH_CREDENTIAL]) {

                    sh '''
                        set -e

                        echo "Stopping old application if running..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} '

                            sudo systemctl stop ${APP_NAME} || true

                            '

                        echo "Starting new application..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} '

                            sudo systemctl start ${APP_NAME}

                            sleep 5

                            sudo systemctl status ${APP_NAME} --no-pager

                            '
                    '''
                }
            }
        }


        // ==========================================================
        // 9. HEALTH CHECK
        // ==========================================================

        stage('Health Check') {

            steps {

                sshagent(credentials: [SSH_CREDENTIAL]) {

                    sh '''
                        set -e

                        echo "Waiting for application..."

                        sleep 5

                        echo "Checking systemd status..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} '

                            sudo systemctl is-active --quiet ${APP_NAME}

                            if [ $? -ne 0 ]
                            then
                                echo "Application service is not running."

                                sudo systemctl status ${APP_NAME} --no-pager

                                sudo journalctl -u ${APP_NAME} \
                                    -n 50 \
                                    --no-pager

                                exit 1
                            fi

                            echo "Application service is running."

                            '

                        echo "Checking port ${APP_PORT}..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} '

                            curl \
                                --fail \
                                --silent \
                                --show-error \
                                http://localhost:${APP_PORT}/ \
                                > /dev/null

                            '

                        echo "========================================"
                        echo "APPLICATION HEALTH CHECK PASSED"
                        echo "========================================"
                    '''
                }
            }
        }
    }


    // ==============================================================
    // POST ACTIONS
    // ==============================================================

    post {

        success {

            echo """
            ========================================
                     CD SUCCESS
            ========================================

            CI Job       : ${CI_JOB}
            CI Build     : ${params.CI_BUILD_NUMBER}

            Application  : ${APP_NAME}
            Server       : ${VM_HOST}
            Port         : ${APP_PORT}

            Deployment completed successfully.

            ========================================
            """
        }

        failure {

            echo """
            ========================================
                     CD FAILED
            ========================================

            CI Job       : ${CI_JOB}
            CI Build     : ${params.CI_BUILD_NUMBER}

            Server       : ${VM_HOST}

            Check the failed stage and console log.

            ========================================
            """
        }

        always {

            echo "CD Pipeline execution completed."
        }
    }
}
