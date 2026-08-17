pipeline {
    agent any

    environment {
        VM_HOST = '43.204.217.49'
        VM_USER = 'ec2-user'

        APP_NAME = 'java-vm-demo'
        TOMCAT_HOME = '/opt/tomcat'

        SSH_CREDENTIALS = 'vm-deploy-key'

        CI_JOB_NAME = 'CI-JOB'
    }

    stages {

        /*
         * Get the WAR produced by CI
         */
        stage('Get CI Artifact') {
            steps {

                copyArtifacts(
                    projectName: "${CI_JOB_NAME}",
                    selector: lastSuccessful(),
                    filter: 'target/*.war'
                )

                sh '''
                    echo "===== CI Artifact ====="
                    find . -name "*.war" -type f
                '''
            }
        }


        /*
         * Connect to Amazon Linux VM
         * Install Java and Tomcat if required
         */
        stage('Prepare Amazon Linux VM') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_HOST} 'bash -s' << 'REMOTE_SCRIPT'

                        set -e

                        echo "======================================"
                        echo "Checking Amazon Linux"
                        echo "======================================"

                        cat /etc/os-release


                        echo "======================================"
                        echo "Checking Java"
                        echo "======================================"

                        if command -v java >/dev/null 2>&1
                        then

                            echo "Java already installed"
                            java -version

                        else

                            echo "Java not installed"
                            echo "Installing Java..."

                            sudo dnf install -y java-17-amazon-corretto

                            echo "Java installation completed"

                            java -version

                        fi


                        echo "======================================"
                        echo "Checking Tomcat"
                        echo "======================================"

                        if [ -f ${TOMCAT_HOME}/bin/catalina.sh ]
                        then

                            echo "Tomcat already installed"

                        else

                            echo "Tomcat not installed"
                            echo "Installing Tomcat..."

                            cd /tmp

                            wget -q \
                            https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.46/bin/apache-tomcat-10.1.46.tar.gz


                            echo "Creating Tomcat directory"

                            sudo mkdir -p ${TOMCAT_HOME}


                            echo "Extracting Tomcat"

                            sudo tar -xzf \
                            apache-tomcat-10.1.46.tar.gz \
                            --strip-components=1 \
                            -C ${TOMCAT_HOME}


                            echo "Creating tomcat user"

                            sudo useradd \
                            --system \
                            --home ${TOMCAT_HOME} \
                            --shell /sbin/nologin \
                            tomcat 2>/dev/null || true


                            echo "Setting permissions"

                            sudo chown -R tomcat:tomcat ${TOMCAT_HOME}

                            sudo chmod +x ${TOMCAT_HOME}/bin/*.sh


                            echo "Creating systemd service"


                            sudo tee /etc/systemd/system/tomcat.service > /dev/null <<EOF

[Unit]
Description=Apache Tomcat
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/java-17-amazon-corretto"
Environment="CATALINA_HOME=${TOMCAT_HOME}"
Environment="CATALINA_BASE=${TOMCAT_HOME}"

ExecStart=${TOMCAT_HOME}/bin/startup.sh
ExecStop=${TOMCAT_HOME}/bin/shutdown.sh

Restart=on-failure

[Install]
WantedBy=multi-user.target

EOF


                            echo "Reloading systemd"

                            sudo systemctl daemon-reload

                            sudo systemctl enable tomcat

                            sudo systemctl start tomcat

                        fi


                        echo "======================================"
                        echo "Tomcat status"
                        echo "======================================"

                        sudo systemctl status tomcat --no-pager || true

                        REMOTE_SCRIPT

                    """
                }
            }
        }


        /*
         * Copy WAR from Jenkins to VM
         */
        stage('Copy WAR to VM') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        echo "======================================"
                        echo "Finding WAR"
                        echo "======================================"

                        WAR_FILE=\\\$(find . -name "*.war" -type f | head -1)


                        if [ -z "\\\$WAR_FILE" ]
                        then

                            echo "ERROR: WAR file not found"

                            exit 1

                        fi


                        echo "WAR found:"
                        echo "\\\$WAR_FILE"


                        echo "======================================"
                        echo "Copying WAR to Amazon Linux"
                        echo "======================================"


                        scp \
                        -o StrictHostKeyChecking=no \
                        "\\\$WAR_FILE" \
                        ${VM_USER}@${VM_HOST}:/tmp/${APP_NAME}.war

                    """
                }
            }
        }


        /*
         * Deploy WAR into Tomcat
         */
        stage('Deploy Application') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        ssh \
                        -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_HOST} '

                            set -e

                            echo "======================================"
                            echo "Stopping Tomcat"
                            echo "======================================"

                            sudo systemctl stop tomcat


                            echo "======================================"
                            echo "Removing old application"
                            echo "======================================"

                            sudo rm -rf \
                            ${TOMCAT_HOME}/webapps/${APP_NAME}

                            sudo rm -f \
                            ${TOMCAT_HOME}/webapps/${APP_NAME}.war


                            echo "======================================"
                            echo "Deploying new WAR"
                            echo "======================================"

                            sudo cp \
                            /tmp/${APP_NAME}.war \
                            ${TOMCAT_HOME}/webapps/${APP_NAME}.war


                            sudo chown \
                            tomcat:tomcat \
                            ${TOMCAT_HOME}/webapps/${APP_NAME}.war


                            echo "======================================"
                            echo "Starting Tomcat"
                            echo "======================================"

                            sudo systemctl start tomcat


                            echo "Waiting for application startup..."

                            sleep 10


                            echo "======================================"
                            echo "Tomcat status"
                            echo "======================================"

                            sudo systemctl status tomcat --no-pager

                        '
                    """
                }
            }
        }


        /*
         * Verify application
         */
        stage('Health Check') {

            steps {

                sh """

                    echo "======================================"
                    echo "Application Health Check"
                    echo "======================================"


                    curl -f \
                    http://${VM_HOST}:8080/${APP_NAME}/


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

            echo '''
            ======================================
            CD SUCCESS
            ======================================

            Application deployed successfully
            to Amazon Linux VM.

            VM:
            43.204.217.49

            Tomcat:
            8080

            ======================================
            '''

        }


        failure {

            echo '''
            ======================================
            CD FAILED
            ======================================

            Please check Jenkins console output.

            ======================================
            '''

        }
    }
}
