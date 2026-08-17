pipeline {

    agent any

    environment {
        CI_JOB = 'Java_CI'

        VM_IP = '43.204.217.49'
        VM_USER = 'ec2-user'

        APP_NAME = 'java-vm-demo'
        APP_DIR = '/opt/java-vm-demo'

        JAR_NAME = 'java-vm-demo.jar'

        TOMCAT_DIR = '/opt/tomcat'
        TOMCAT_PORT = '8080'
    }

    stages {

        stage('Download Artifact From CI') {

            steps {

                echo "Downloading artifact from CI job: ${CI_JOB}"

                copyArtifacts(
                    projectName: "${CI_JOB}",
                    selector: lastSuccessful(),
                    filter: 'target/*.jar',
                    flatten: true
                )
            }
        }

        stage('Verify Artifact') {

            steps {

                sh '''
                    echo "Files in workspace:"
                    ls -lh

                    echo "Checking JAR..."

                    if [ ! -f *.jar ]; then
                        echo "ERROR: JAR file not found"
                        exit 1
                    fi

                    ls -lh *.jar
                '''
            }
        }

        stage('Test SSH Connection') {

            steps {

                sshagent(['aws-vm-key']) {

                    sh """
                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_IP} \
                        'echo SSH connection successful'
                    """
                }
            }
        }

        stage('Install Java 17') {

            steps {

                sshagent(['aws-vm-key']) {

                    sh """

                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_IP} << 'EOF'

                        set -e

                        echo "Checking Java..."

                        if java -version 2>&1 | grep -q '17'; then
                            echo "Java 17 already installed"
                        else

                            echo "Installing Java 17..."

                            sudo dnf install -y java-17-amazon-corretto

                        fi

                        java -version

                        EOF

                    """
                }
            }
        }

        stage('Install Tomcat') {

            steps {

                sshagent(['aws-vm-key']) {

                    sh """

                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_IP} << 'EOF'

                        set -e

                        if [ -d "${TOMCAT_DIR}" ]; then

                            echo "Tomcat already installed"

                        else

                            echo "Installing Tomcat..."

                            sudo mkdir -p ${TOMCAT_DIR}

                            cd /tmp

                            curl -L -o tomcat.tar.gz \
                            https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.46/bin/apache-tomcat-10.1.46.tar.gz

                            sudo tar -xzf tomcat.tar.gz \
                            --strip-components=1 \
                            -C ${TOMCAT_DIR}

                            sudo chown -R ${VM_USER}:${VM_USER} ${TOMCAT_DIR}

                        fi

                        echo "Tomcat installation completed"

                        EOF

                    """
                }
            }
        }

        stage('Prepare Application') {

            steps {

                sshagent(['aws-vm-key']) {

                    sh """

                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_IP} << 'EOF'

                        sudo mkdir -p ${APP_DIR}

                        sudo chown -R ${VM_USER}:${VM_USER} ${APP_DIR}

                        EOF

                    """
                }
            }
        }

        stage('Copy JAR') {

            steps {

                sshagent(['aws-vm-key']) {

                    sh """

                        scp -o StrictHostKeyChecking=no \
                        *.jar \
                        ${VM_USER}@${VM_IP}:${APP_DIR}/${JAR_NAME}

                    """
                }
            }
        }

        stage('Deploy Application') {

            steps {

                sshagent(['aws-vm-key']) {

                    sh """

                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_IP} << 'EOF'

                        set -e

                        echo "Stopping old application..."

                        sudo pkill -f "${JAR_NAME}" || true

                        sleep 5

                        echo "Starting new application..."

                        nohup java -jar \
                        ${APP_DIR}/${JAR_NAME} \
                        --server.port=${TOMCAT_PORT} \
                        > ${APP_DIR}/application.log 2>&1 &

                        echo \$! > ${APP_DIR}/application.pid

                        sleep 10

                        echo "Application started"

                        cat ${APP_DIR}/application.pid

                        EOF

                    """
                }
            }
        }

        stage('Health Check') {

            steps {

                sshagent(['aws-vm-key']) {

                    sh """

                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_IP} << 'EOF'

                        echo "Checking application..."

                        sleep 5

                        curl -f http://localhost:${TOMCAT_PORT}/ || {

                            echo "Application health check failed"

                            echo "Application logs:"
                            tail -100 ${APP_DIR}/application.log

                            exit 1
                        }

                        echo "Application is UP"

                        EOF

                    """
                }
            }
        }
    }

    post {

        success {

            echo '''
            ========================================
            CD SUCCESS
            ========================================
            Application deployed successfully.
            '''

        }

        failure {

            echo '''
            ========================================
            CD FAILED
            ========================================
            Check the failed stage.
            '''

        }
    }
}
