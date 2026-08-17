pipeline {

    agent any

    environment {
        CI_JOB  = 'Java_CI_Project'

        VM_IP   = '43.204.217.49'
        VM_USER = 'ec2-user'

        APP_DIR  = '/opt/java-vm-demo'
        JAR_NAME = 'java-vm-demo.jar'
        APP_PORT = '8080'
    }

    stages {

        stage('Download Artifact From CI') {
            steps {
                echo "Downloading artifact from: ${CI_JOB}"

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
                    echo "Checking downloaded artifact..."

                    ls -lh

                    JAR_FILE=$(find . -maxdepth 1 -name "*.jar" | head -1)

                    if [ -z "$JAR_FILE" ]; then
                        echo "ERROR: JAR file not found"
                        exit 1
                    fi

                    echo "Artifact found:"
                    ls -lh "$JAR_FILE"
                '''
            }
        }

        stage('Test SSH Connection') {
            steps {
                sshagent(['aws-vm-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_IP} \
                        "echo SSH connection successful"
                    '''
                }
            }
        }

        stage('Install Java 17') {
            steps {
                sshagent(['aws-vm-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_IP} << 'EOF'

                        set -e

                        echo "Checking Java..."

                        if java -version 2>&1 | grep -q '"17'; then
                            echo "Java 17 already installed"
                        else
                            echo "Installing Java 17..."
                            sudo dnf install -y java-17-amazon-corretto
                        fi

                        echo "Java version:"
                        java -version

                        EOF
                    '''
                }
            }
        }

        stage('Prepare Application Directory') {
            steps {
                sshagent(['aws-vm-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_IP} << 'EOF'

                        sudo mkdir -p ${APP_DIR}
                        sudo chown -R ${VM_USER}:${VM_USER} ${APP_DIR}

                        echo "Application directory ready"

                        EOF
                    '''
                }
            }
        }

        stage('Copy JAR To VM') {
            steps {
                sshagent(['aws-vm-key']) {
                    sh '''
                        JAR_FILE=$(find . -maxdepth 1 -name "*.jar" | head -1)

                        echo "Copying $JAR_FILE to VM..."

                        scp -o StrictHostKeyChecking=no \
                        "$JAR_FILE" \
                        ${VM_USER}@${VM_IP}:${APP_DIR}/${JAR_NAME}
                    '''
                }
            }
        }

        stage('Deploy Application') {
            steps {
                sshagent(['aws-vm-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_IP} << 'EOF'

                        set -e

                        echo "Stopping previous application..."

                        if [ -f ${APP_DIR}/application.pid ]; then

                            PID=$(cat ${APP_DIR}/application.pid)

                            if kill -0 $PID 2>/dev/null; then
                                kill $PID
                                sleep 5
                            fi

                            rm -f ${APP_DIR}/application.pid

                        fi

                        echo "Starting new application..."

                        nohup java -jar \
                        ${APP_DIR}/${JAR_NAME} \
                        --server.port=${APP_PORT} \
                        > ${APP_DIR}/application.log 2>&1 &

                        echo $! > ${APP_DIR}/application.pid

                        echo "Application PID:"
                        cat ${APP_DIR}/application.pid

                        sleep 10

                        echo "Application started"

                        EOF
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                sshagent(['aws-vm-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_IP} << 'EOF'

                        echo "Checking application..."

                        if curl -f http://localhost:${APP_PORT}/; then
                            echo "Application is UP"
                        else
                            echo "Application health check FAILED"
                            echo "Application logs:"
                            tail -100 ${APP_DIR}/application.log
                            exit 1
                        fi

                        EOF
                    '''
                }
            }
        }
    }

    post {

        success {
            echo '''
            ==========================================
                    CD DEPLOYMENT SUCCESS
            ==========================================

            Application deployed successfully.

            VM       : 43.204.217.49
            Port     : 8080
            ==========================================
            '''
        }

        failure {
            echo '''
            ==========================================
                    CD DEPLOYMENT FAILED
            ==========================================

            Check the failed stage and console logs.
            ==========================================
            '''
        }
    }
}
