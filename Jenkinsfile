pipeline {

    agent any

    environment {

        // IMPORTANT: Put your EXACT Jenkins CI job name here
        CI_JOB = 'Java_cd_test'

        VM_IP = '43.204.217.49'
        VM_USER = 'ec2-user'

        APP_DIR = '/opt/java-vm-demo'
        JAR_NAME = 'java-vm-demo.jar'
    }

    stages {

        stage('Download Artifact From CI') {

            steps {

                echo "Downloading artifact from CI: ${CI_JOB}"

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
                    echo "Workspace contents:"
                    ls -lh

                    echo "Checking JAR..."

                    JAR_FILE=$(find . -maxdepth 1 -name "*.jar" | head -1)

                    if [ -z "$JAR_FILE" ]; then
                        echo "ERROR: No JAR found"
                        exit 1
                    fi

                    echo "JAR found:"
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

                        java -version

                        EOF
                    '''
                }
            }
        }

        stage('Prepare Application') {

            steps {

                sshagent(['aws-vm-key']) {

                    sh '''

                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_IP} << 'EOF'

                        sudo mkdir -p ${APP_DIR}

                        sudo chown -R ${VM_USER}:${VM_USER} ${APP_DIR}

                        EOF
                    '''
                }
            }
        }

        stage('Copy JAR') {

            steps {

                sshagent(['aws-vm-key']) {

                    sh '''

                        JAR_FILE=$(find . -maxdepth 1 -name "*.jar" | head -1)

                        echo "Copying: $JAR_FILE"

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

                        echo "Stopping old application..."

                        if [ -f ${APP_DIR}/application.pid ]; then

                            PID=$(cat ${APP_DIR}/application.pid)

                            sudo kill $PID || true

                            rm -f ${APP_DIR}/application.pid

                            sleep 5

                        fi

                        echo "Starting application..."

                        nohup java -jar \
                        ${APP_DIR}/${JAR_NAME} \
                        --server.port=8080 \
                        > ${APP_DIR}/application.log 2>&1 &

                        echo $! > ${APP_DIR}/application.pid

                        echo "Application PID:"
                        cat ${APP_DIR}/application.pid

                        sleep 10

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

                        sleep 5

                        curl -f http://localhost:8080/ || {

                            echo "Application health check failed"

                            echo "Application logs:"

                            tail -100 ${APP_DIR}/application.log

                            exit 1
                        }

                        echo "Application is UP"

                        EOF
                    '''
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
