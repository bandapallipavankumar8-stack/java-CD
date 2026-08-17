pipeline {

    agent any

    environment {

        VM_HOST = '43.204.217.49'
        VM_USER = 'ec2-user'

        APP_NAME = 'java-vm-demo'

        TOMCAT_HOME = '/opt/tomcat'

        SSH_CREDENTIALS = 'vm-deploy-key'

        CI_JOB = 'java-vm-demo-CI'
    }

    stages {

        /*
         * 1. Get WAR produced by CI
         */
        stage('Get Artifact From CI') {

            steps {

                copyArtifacts(
                    projectName: "${CI_JOB}",
                    selector: lastSuccessful(),
                    filter: 'target/*.war'
                )

                sh '''
                    echo "CI artifact:"
                    find . -name "*.war" -type f -ls
                '''
            }
        }


        /*
         * 2. Install Java and Tomcat
         */
        stage('Prepare VM') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        ssh -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_HOST} 'bash -s' << 'REMOTE'

                        set -e

                        echo "================================="
                        echo "Checking Java"
                        echo "================================="


                        if command -v java >/dev/null 2>&1
                        then

                            echo "Java already installed"
                            java -version

                        else

                            echo "Java not installed"
                            echo "Installing Java 17..."

                            sudo dnf install -y java-17-amazon-corretto

                            java -version

                        fi


                        echo "================================="
                        echo "Checking Tomcat"
                        echo "================================="


                        if [ -f ${TOMCAT_HOME}/bin/catalina.sh ]
                        then

                            echo "Tomcat already installed"

                        else

                            echo "Installing Tomcat..."

                            cd /tmp

                            wget -q \
                            https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.46/bin/apache-tomcat-10.1.46.tar.gz


                            sudo mkdir -p ${TOMCAT_HOME}


                            sudo tar -xzf \
                            apache-tomcat-10.1.46.tar.gz \
                            --strip-components=1 \
                            -C ${TOMCAT_HOME}


                            echo "Creating Tomcat user"

                            sudo useradd \
                            --system \
                            --home ${TOMCAT_HOME} \
                            --shell /sbin/nologin \
                            tomcat 2>/dev/null || true


                            sudo chown -R \
                            tomcat:tomcat \
                            ${TOMCAT_HOME}


                            sudo chmod +x \
                            ${TOMCAT_HOME}/bin/*.sh


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


                            sudo systemctl daemon-reload

                            sudo systemctl enable tomcat

                            sudo systemctl start tomcat

                        fi


                        echo "================================="
                        echo "Tomcat status"
                        echo "================================="

                        sudo systemctl status tomcat --no-pager || true


                        REMOTE

                    """
                }
            }
        }


        /*
         * 3. Copy WAR from Jenkins to VM
         */
        stage('Copy WAR') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        WAR_FILE=\\\$(find . -name "*.war" -type f | head -1)


                        if [ -z "\\\$WAR_FILE" ]
                        then

                            echo "ERROR: WAR file not found"

                            exit 1

                        fi


                        echo "WAR:"
                        echo "\\\$WAR_FILE"


                        scp \
                        -o StrictHostKeyChecking=no \
                        "\\\$WAR_FILE" \
                        ${VM_USER}@${VM_HOST}:/tmp/${APP_NAME}.war

                    """
                }
            }
        }


        /*
         * 4. Deploy application
         */
        stage('Deploy Application') {

            steps {

                sshagent(["${SSH_CREDENTIALS}"]) {

                    sh """

                        ssh \
                        -o StrictHostKeyChecking=no \
                        ${VM_USER}@${VM_HOST} '

                            set -e

                            echo "Stopping Tomcat"

                            sudo systemctl stop tomcat


                            echo "Removing old deployment"

                            sudo rm -rf \
                            ${TOMCAT_HOME}/webapps/${APP_NAME}

                            sudo rm -f \
                            ${TOMCAT_HOME}/webapps/${APP_NAME}.war


                            echo "Copying new WAR"

                            sudo cp \
                            /tmp/${APP_NAME}.war \
                            ${TOMCAT_HOME}/webapps/${APP_NAME}.war


                            sudo chown \
                            tomcat:tomcat \
                            ${TOMCAT_HOME}/webapps/${APP_NAME}.war


                            echo "Starting Tomcat"

                            sudo systemctl start tomcat


                            echo "Waiting for application..."

                            sleep 10


                            sudo systemctl status tomcat --no-pager

                        '
                    """
                }
            }
        }


        /*
         * 5. Health check
         */
        stage('Health Check') {

            steps {

                sh """

                    echo "Checking application..."

                    curl -f \
                    http://${VM_HOST}:8080/${APP_NAME}/

                    echo ""

                    echo "================================="
                    echo "DEPLOYMENT SUCCESSFUL"
                    echo "================================="

                """
            }
        }
    }


    post {

        success {

            echo "CD SUCCESS: Application deployed to ${VM_HOST}"

        }

        failure {

            echo "CD FAILED: Check Jenkins console logs."

        }
    }
}
