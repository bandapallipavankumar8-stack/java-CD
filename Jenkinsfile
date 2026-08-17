pipeline {

    agent any

    environment {

        // ==============================
        // CI JOB
        // ==============================
        CI_JOB = 'Java_CI'

        // ==============================
        // VM DETAILS
        // ==============================
        VM_HOST = '43.204.217.49'
        VM_USER = 'ec2-user'

        // ==============================
        // APPLICATION
        // ==============================
        APP_NAME = 'java-vm-demo'
        APP_DIR = '/opt/java-vm-demo'

        // ==============================
        // JENKINS SSH CREDENTIAL
        // ==============================
        SSH_CREDENTIALS = 'vm-deploy-key'
    }


    stages {


        // =====================================================
        // 1. DOWNLOAD ARTIFACT FROM CI
        // =====================================================

        stage('Download Artifact From CI') {

            steps {

                echo "Downloading artifact from CI job: ${CI_JOB}"

                copyArtifacts(
                    projectName: "${CI_JOB}",
                    selector: lastSuccessfulBuild(),
                    filter: 'target/*.jar',
                    fingerprintArtifacts: true
                )


                sh '''
                    echo "======================================"
                    echo "Downloaded JAR files"
                    echo "======================================"

                    find . -type f -name "*.jar" -ls
                '''
            }
        }


        // =====================================================
        // 2. VERIFY ARTIFACT
        // =====================================================

        stage('Verify Artifact') {

            steps {

                sh '''

                    echo "Checking JAR..."

                    JAR_FILE=$(find target -name "*.jar" -type f | head -1)

                    if [ -z "$JAR_FILE" ]
                    then

                        echo "ERROR: JAR file not found"

                        exit 1

                    fi

                    echo "JAR found:"
                    echo "$JAR_FILE"

                    ls -lh "$JAR_FILE"

                '''
            }
        }


        // =====================================================
        // 3. TEST SSH CONNECTION
        // =====================================================

        stage('Test SSH Connection') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        echo "Testing SSH connection..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            -o ConnectTimeout=10 \
                            ${VM_USER}@${VM_HOST} \
                            'hostname && whoami && echo SSH connection successful'

                    """
                }
            }
        }


        // =====================================================
        // 4. INSTALL JAVA
        // =====================================================

        stage('Install Java') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} \
                            'bash -s' << 'REMOTE'

                            set -e

                            echo "======================================"
                            echo "Checking Java"
                            echo "======================================"


                            if command -v java >/dev/null 2>&1
                            then

                                echo "Java already installed"

                                java -version

                            else

                                echo "Java not installed"

                                echo "Installing Amazon Corretto 17..."

                                sudo dnf install -y java-17-amazon-corretto

                                echo "Java installation completed"

                                java -version

                            fi

                        REMOTE

                    """
                }
            }
        }


        // =====================================================
        // 5. PREPARE APPLICATION DIRECTORY
        // =====================================================

        stage('Prepare Application') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} \
                            'bash -s' << 'REMOTE'

                            set -e

                            echo "======================================"
                            echo "Preparing application"
                            echo "======================================"


                            sudo mkdir -p ${APP_DIR}


                            if id ${APP_NAME} >/dev/null 2>&1
                            then

                                echo "Application user already exists"

                            else

                                echo "Creating application user"

                                sudo useradd \
                                    --system \
                                    --shell /sbin/nologin \
                                    ${APP_NAME}

                            fi


                            sudo chown -R \
                                ${APP_NAME}:${APP_NAME} \
                                ${APP_DIR}


                            echo "Application directory ready"

                            sudo ls -ld ${APP_DIR}

                        REMOTE

                    """
                }
            }
        }


        // =====================================================
        // 6. COPY JAR TO VM
        // =====================================================

        stage('Copy Application') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        JAR_FILE=\\\$(find target -name "*.jar" -type f | head -1)


                        echo "======================================"
                        echo "Copying application"
                        echo "======================================"

                        echo "JAR:"
                        echo "\\\$JAR_FILE"


                        scp \
                            -o StrictHostKeyChecking=no \
                            "\\\$JAR_FILE" \
                            ${VM_USER}@${VM_HOST}:/tmp/${APP_NAME}.jar

                    """
                }
            }
        }


        // =====================================================
        // 7. DEPLOY APPLICATION
        // =====================================================

        stage('Deploy Application') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} \
                            'bash -s' << 'REMOTE'

                            set -e

                            echo "======================================"
                            echo "Deploying application"
                            echo "======================================"


                            sudo cp \
                                /tmp/${APP_NAME}.jar \
                                ${APP_DIR}/${APP_NAME}.jar


                            sudo chown \
                                ${APP_NAME}:${APP_NAME} \
                                ${APP_DIR}/${APP_NAME}.jar


                            sudo chmod 755 \
                                ${APP_DIR}/${APP_NAME}.jar


                            echo "Application copied successfully"


                            sudo ls -lh \
                                ${APP_DIR}/${APP_NAME}.jar

                        REMOTE

                    """
                }
            }
        }


        // =====================================================
        // 8. CREATE SYSTEMD SERVICE
        // =====================================================

        stage('Configure Systemd') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} \
                            'bash -s' << 'REMOTE'

                            set -e

                            echo "======================================"
                            echo "Creating systemd service"
                            echo "======================================"


                            sudo tee /etc/systemd/system/${APP_NAME}.service > /dev/null <<EOF

[Unit]
Description=${APP_NAME} Spring Boot Application
After=network.target

[Service]

Type=simple

User=${APP_NAME}
Group=${APP_NAME}

WorkingDirectory=${APP_DIR}

ExecStart=/usr/bin/java -jar ${APP_DIR}/${APP_NAME}.jar

Restart=always
RestartSec=5

SuccessExitStatus=143

[Install]

WantedBy=multi-user.target

EOF


                            echo "Reloading systemd"

                            sudo systemctl daemon-reload


                            echo "Enabling application"

                            sudo systemctl enable ${APP_NAME}


                        REMOTE

                    """
                }
            }
        }


        // =====================================================
        // 9. RESTART APPLICATION
        // =====================================================

        stage('Restart Application') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} \
                            'bash -s' << 'REMOTE'

                            set -e

                            echo "======================================"
                            echo "Starting application"
                            echo "======================================"


                            sudo systemctl restart ${APP_NAME}


                            echo "Waiting for application..."

                            sleep 10


                            echo "======================================"
                            echo "Service status"
                            echo "======================================"


                            sudo systemctl status \
                                ${APP_NAME} \
                                --no-pager


                        REMOTE

                    """
                }
            }
        }


        // =====================================================
        // 10. HEALTH CHECK
        // =====================================================

        stage('Health Check') {

            steps {

                sh """

                    echo "======================================"
                    echo "Application Health Check"
                    echo "======================================"


                    sleep 5


                    curl \
                        --fail \
                        --show-error \
                        --silent \
                        http://${VM_HOST}:8080/ \
                        > /dev/null


                    echo "Application is UP"


                    echo "======================================"
                    echo "DEPLOYMENT SUCCESSFUL"
                    echo "======================================"

                """
            }
        }
    }


    // =========================================================
    // POST ACTIONS
    // =========================================================

    post {

        success {

            echo """
            ======================================
            CD SUCCESS
            ======================================

            Application : ${APP_NAME}
            VM          : ${VM_HOST}
            Port        : 8080

            Application successfully deployed.
            """

        }


        failure {

            echo """
            ======================================
            CD FAILED
            ======================================

            Check the failed stage above.
            """

        }


        always {

            echo "CD Pipeline completed."

        }
    }
}
