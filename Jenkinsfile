pipeline {

    agent any

    environment {
        VM_HOST = '43.204.217.49'
        VM_USER = 'ec2-user'

        APP_NAME = 'java-vm-demo'
        APP_DIR = '/opt/java-vm-demo'

        SSH_CREDENTIALS = 'vm-deploy-key'

        CI_JOB = 'Java_CI'
    }

    stages {

        stage('Download Artifact From CI') {
            steps {

                copyArtifacts(
                    projectName: "${CI_JOB}",
                    selector: lastSuccessful(),
                    filter: 'target/*.jar'
                )

                sh '''
                    echo "===== Artifact ====="

                    find . -name "*.jar" -type f -ls
                '''
            }
        }


        stage('Prepare Amazon Linux VM') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_HOST} 'bash -s' << 'REMOTE'

                        set -e

                        echo "======================================"
                        echo "Checking Java"
                        echo "======================================"

                        if command -v java >/dev/null 2>&1
                        then
                            echo "Java already installed"
                            java -version

                        else
                            echo "Installing Java 17"

                            sudo dnf install -y java-17-amazon-corretto

                            java -version
                        fi


                        echo "======================================"
                        echo "Creating application directory"
                        echo "======================================"

                        sudo mkdir -p ${APP_DIR}

                        sudo useradd \
                            --system \
                            --shell /sbin/nologin \
                            ${APP_NAME} 2>/dev/null || true

                        sudo chown -R \
                            ${APP_NAME}:${APP_NAME} \
                            ${APP_DIR}


                        echo "======================================"
                        echo "Creating systemd service"
                        echo "======================================"

                        sudo tee /etc/systemd/system/${APP_NAME}.service > /dev/null <<EOF

[Unit]
Description=Java VM Demo Application
After=network.target

[Service]
User=${APP_NAME}
Group=${APP_NAME}

WorkingDirectory=${APP_DIR}

ExecStart=/usr/bin/java -jar ${APP_DIR}/${APP_NAME}.jar

Restart=always
RestartSec=5

Environment="JAVA_OPTS=-Xms256m -Xmx512m"

[Install]
WantedBy=multi-user.target

EOF


                        sudo systemctl daemon-reload

                        sudo systemctl enable ${APP_NAME}

                        REMOTE

                    """
                }
            }
        }


        stage('Deploy Application') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        echo "======================================"
                        echo "Finding JAR"
                        echo "======================================"

                        JAR_FILE=\\\$(find . -name "*.jar" -type f | head -1)


                        if [ -z "\\\$JAR_FILE" ]
                        then

                            echo "ERROR: JAR file not found"

                            exit 1

                        fi


                        echo "Deploying: \\\$JAR_FILE"


                        echo "======================================"
                        echo "Copying JAR to VM"
                        echo "======================================"


                        scp \
                            -o StrictHostKeyChecking=no \
                            "\\\$JAR_FILE" \
                            ${VM_USER}@${VM_HOST}:/tmp/${APP_NAME}.jar


                        echo "======================================"
                        echo "Installing application"
                        echo "======================================"


                        ssh \
                            -o StrictHostKeyChecking=no \
                            ${VM_USER}@${VM_HOST} '

                            set -e

                            sudo cp \
                                /tmp/${APP_NAME}.jar \
                                ${APP_DIR}/${APP_NAME}.jar

                            sudo chown \
                                ${APP_NAME}:${APP_NAME} \
                                ${APP_DIR}/${APP_NAME}.jar

                            sudo systemctl restart ${APP_NAME}

                            sleep 10

                            sudo systemctl status \
                                ${APP_NAME} \
                                --no-pager

                            '

                    """
                }
            }
        }


        stage('Health Check') {

            steps {

                sh """

                    echo "======================================"
                    echo "Health Check"
                    echo "======================================"


                    curl -f \
                        http://${VM_HOST}:8080/


                    echo ""

                    echo "======================================"
                    echo "DEPLOYMENT SUCCESSFUL"
                    echo "======================================"

                """
            }
        }
    }


    post {

        success {
            echo "CD SUCCESS"
        }

        failure {
            echo "CD FAILED"
        }
    }
}
