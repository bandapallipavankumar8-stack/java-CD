pipeline {

    agent any

    parameters {
        string(
            name: 'CI_BUILD_NUMBER',
            defaultValue: '1',
            description: 'Successful CI build number to deploy'
        )
    }

    environment {

        CI_JOB = 'Java_CI'

        VM_HOST = '43.204.217.49'
        VM_USER = 'ec2-user'

        APP_NAME = 'java-vm-demo'
        APP_DIR = '/opt/java-vm-demo'

        SSH_CREDENTIALS = 'vm-deploy-key'
    }

    stages {

        stage('Download Artifact From CI') {
            steps {

                echo "Downloading artifact from CI"
                echo "CI Job     : ${CI_JOB}"
                echo "Build      : ${CI_BUILD_NUMBER}"

                copyArtifacts(
                    projectName: "${CI_JOB}",
                    selector: specific("${CI_BUILD_NUMBER}"),
                    filter: 'target/*.jar',
                    fingerprintArtifacts: true
                )

                sh '''
                    echo "Downloaded artifacts:"
                    find target -type f -name "*.jar" -ls
                '''
            }
        }

        stage('Verify Artifact') {
            steps {

                sh '''
                    JAR_FILE=$(find target -type f -name "*.jar" | head -1)

                    if [ -z "$JAR_FILE" ]; then
                        echo "ERROR: JAR file not found"
                        exit 1
                    fi

                    echo "JAR found:"
                    ls -lh "$JAR_FILE"
                '''
            }
        }

        stage('Test SSH Connection') {
            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """
                        ssh \
                        -o StrictHostKeyChecking=no \
                        -o ConnectTimeout=10 \
                        ${VM_USER}@${VM_HOST} \
                        'hostname && whoami && java -version || true'
                    """
                }
            }
        }

        stage('Install Java 17') {
            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """
                        ssh \
                        -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_HOST} \
                        'bash -s' << 'REMOTE'

set -e

echo "Checking Java..."

if command -v java >/dev/null 2>&1; then

    echo "Java already installed"
    java -version

else

    echo "Installing Java 17..."

    sudo dnf install -y java-17-amazon-corretto

    java -version

fi

REMOTE
                    """
                }
            }
        }

        stage('Prepare Application') {
            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """
                        ssh \
                        -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_HOST} \
                        'bash -s' << 'REMOTE'

set -e

sudo mkdir -p ${APP_DIR}

if ! id ${APP_NAME} >/dev/null 2>&1; then
    sudo useradd --system --shell /sbin/nologin ${APP_NAME}
fi

sudo chown -R ${APP_NAME}:${APP_NAME} ${APP_DIR}

REMOTE
                    """
                }
            }
        }

        stage('Copy JAR') {
            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh '''
                        JAR_FILE=$(find target -type f -name "*.jar" | head -1)

                        echo "Copying:"
                        echo "$JAR_FILE"

                        scp \
                        -o StrictHostKeyChecking=no \
                        "$JAR_FILE" \
                        ${VM_USER}@${VM_HOST}:/tmp/${APP_NAME}.jar
                    '''
                }
            }
        }

        stage('Deploy Application') {
            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """
                        ssh \
                        -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_HOST} \
                        'bash -s' << 'REMOTE'

set -e

sudo mv \
    /tmp/${APP_NAME}.jar \
    ${APP_DIR}/${APP_NAME}.jar

sudo chown \
    ${APP_NAME}:${APP_NAME} \
    ${APP_DIR}/${APP_NAME}.jar

sudo chmod 755 \
    ${APP_DIR}/${APP_NAME}.jar

REMOTE
                    """
                }
            }
        }

        stage('Configure Systemd') {
            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """
                        ssh \
                        -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_HOST} \
                        'bash -s' << 'REMOTE'

set -e

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

sudo systemctl daemon-reload
sudo systemctl enable ${APP_NAME}

REMOTE
                    """
                }
            }
        }

        stage('Restart Application') {
            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """
                        ssh \
                        -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_HOST} \
                        'sudo systemctl restart ${APP_NAME}'
                    """

                    sleep 10
                }
            }
        }

        stage('Health Check') {
            steps {

                sh """

                    echo "Checking application..."

                    curl \
                    --fail \
                    --show-error \
                    --silent \
                    http://${VM_HOST}:8080/ \
                    > /dev/null

                    echo "Application is UP"
                """
            }
        }
    }

    post {

        success {
            echo """
========================================
CD SUCCESS
========================================
Application : ${APP_NAME}
VM          : ${VM_HOST}
Port        : 8080
CI Build    : ${CI_BUILD_NUMBER}
========================================
"""
        }

        failure {
            echo """
========================================
CD FAILED
========================================

Check the failed stage.
"""
        }
    }
}
