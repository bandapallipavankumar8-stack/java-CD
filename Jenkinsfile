pipeline {

    agent any

    environment {

        CI_JOB        = 'Java_CI_Project'
        SERVER_IP     = '43.204.217.49'

        APP_NAME      = 'java-vm-demo'
        JAR_NAME      = 'java-vm-demo-1.0.jar'

        REMOTE_DIR    = '/opt/java-vm-demo'
        REMOTE_JAR    = '/opt/java-vm-demo/java-vm-demo.jar'

        SERVICE_NAME  = 'java-vm-demo'

        APP_PORT      = '8080'

        SSH_CRED_ID   = 'vm-ssh-key'
    }

    stages {

        stage('Download Artifact From CI') {

            steps {

                script {

                    echo """
                    ========================================
                    DOWNLOADING ARTIFACT
                    ========================================

                    CI Job : ${CI_JOB}
                    Artifact: target/*.jar

                    ========================================
                    """

                    copyArtifacts(
                        projectName: env.CI_JOB,
                        selector: lastSuccessful(),
                        filter: 'target/*.jar',
                        flatten: true
                    )
                }
            }
        }


        stage('Verify Artifact') {

            steps {

                sh '''
                    set -e

                    echo "Checking downloaded artifact..."

                    ls -lah

                    if [ ! -f *.jar ]; then
                        echo "ERROR: JAR file not found"
                        exit 1
                    fi

                    echo "Artifact found:"
                    ls -lh *.jar
                '''
            }
        }


        stage('Test SSH Connection') {

            steps {

                sshagent(credentials: [env.SSH_CRED_ID]) {

                    sh '''
                        set -e

                        echo "Testing SSH connection..."

                        ssh -o StrictHostKeyChecking=no \
                            -o ConnectTimeout=10 \
                            ubuntu@${SERVER_IP} \
                            "echo SSH connection successful"

                    '''
                }
            }
        }


        stage('Install Java 17') {

            steps {

                sshagent(credentials: [env.SSH_CRED_ID]) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${SERVER_IP} << 'EOF'

                        set -e

                        echo "Checking Java..."

                        if java -version 2>&1 | grep -q '"17\\.'; then
                            echo "Java 17 already installed"
                        else
                            echo "Installing Java 17..."

                            sudo apt-get update -y
                            sudo apt-get install -y openjdk-17-jdk

                            echo "Java installation completed"
                        fi

                        java -version

                        EOF
                    '''
                }
            }
        }


        stage('Prepare Application') {

            steps {

                sshagent(credentials: [env.SSH_CRED_ID]) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${SERVER_IP} << 'EOF'

                        set -e

                        echo "Creating application directory..."

                        sudo mkdir -p ${REMOTE_DIR}

                        sudo chown -R ubuntu:ubuntu ${REMOTE_DIR}

                        EOF
                    '''
                }
            }
        }


        stage('Copy JAR') {

            steps {

                sshagent(credentials: [env.SSH_CRED_ID]) {

                    sh '''
                        set -e

                        echo "Copying JAR to VM..."

                        JAR_FILE=$(find . -maxdepth 1 -name "*.jar" -type f | head -1)

                        if [ -z "$JAR_FILE" ]; then
                            echo "ERROR: JAR not found"
                            exit 1
                        fi

                        echo "Uploading: $JAR_FILE"

                        scp -o StrictHostKeyChecking=no \
                            "$JAR_FILE" \
                            ubuntu@${SERVER_IP}:${REMOTE_JAR}

                    '''
                }
            }
        }


        stage('Configure Systemd') {

            steps {

                sshagent(credentials: [env.SSH_CRED_ID]) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${SERVER_IP} << 'EOF'

                        set -e

                        echo "Creating systemd service..."

                        sudo tee /etc/systemd/system/${SERVICE_NAME}.service > /dev/null << SERVICE

[Unit]
Description=Java VM Demo Application
After=network.target

[Service]
User=ubuntu
WorkingDirectory=${REMOTE_DIR}
ExecStart=/usr/bin/java -jar ${REMOTE_JAR}
SuccessExitStatus=143
Restart=always
RestartSec=5

Environment=JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64

[Install]
WantedBy=multi-user.target

SERVICE

                        sudo systemctl daemon-reload

                        sudo systemctl enable ${SERVICE_NAME}

                        echo "Systemd configuration completed"

                        EOF
                    '''
                }
            }
        }


        stage('Deploy Application') {

            steps {

                sshagent(credentials: [env.SSH_CRED_ID]) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${SERVER_IP} << 'EOF'

                        set -e

                        echo "Deploying application..."

                        sudo systemctl restart ${SERVICE_NAME}

                        sleep 5

                        sudo systemctl status ${SERVICE_NAME} --no-pager

                        EOF
                    '''
                }
            }
        }


        stage('Health Check') {

            steps {

                sshagent(credentials: [env.SSH_CRED_ID]) {

                    sh '''
                        set -e

                        echo "Performing application health check..."

                        for i in {1..12}
                        do
                            echo "Health check attempt: $i"

                            if curl -f http://${SERVER_IP}:${APP_PORT}/ > /dev/null 2>&1
                            then
                                echo "========================================"
                                echo "APPLICATION IS UP"
                                echo "========================================"

                                exit 0
                            fi

                            sleep 5
                        done

                        echo "========================================"
                        echo "APPLICATION HEALTH CHECK FAILED"
                        echo "========================================"

                        ssh -o StrictHostKeyChecking=no ubuntu@${SERVER_IP} \
                            "sudo journalctl -u ${SERVICE_NAME} -n 100 --no-pager"

                        exit 1
                    '''
                }
            }
        }
    }


    post {

        success {

            echo """
            ========================================
                     CD SUCCESSFUL
            ========================================

            CI Job       : ${CI_JOB}

            Server       : ${SERVER_IP}

            Application  : ${SERVICE_NAME}

            Port         : ${APP_PORT}

            Status       : DEPLOYED SUCCESSFULLY

            ========================================
            """
        }

        failure {

            echo """
            ========================================
                     CD FAILED
            ========================================

            CI Job       : ${CI_JOB}

            Server       : ${SERVER_IP}

            Check the failed stage and console log.

            ========================================
            """
        }

        always {

            echo "CD Pipeline execution completed."
        }
    }
}
