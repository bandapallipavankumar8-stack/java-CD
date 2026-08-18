pipeline {
    agent any
    
    // Tools are required to compile the code on Jenkins
    tools {
        jdk 'jdk' 
        maven 'Maven' 
    }
    
    environment {
        TARGET_IP = '15.207.248.30'
        SERVER_USER = 'ec2-user' 
        SSH_KEY = credentials('target-server-ssh-key')
    }
    
    stages {
        // Added the build stage back to compile your source code into a JAR file
        stage('Build & Package Code') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('Deploy Directly to Target Machine') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'target-server-ssh-key', keyFileVariable: 'KEY_FILE')]) {
                    sh """
                        echo "=== Step 1: Stopping old Java applications ==="
                        ssh -i \$KEY_FILE -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "pkill -f '\\.jar'" || true
                        sleep 2
                        
                        echo "=== Step 2: Uploading JAR directly from Jenkins to Target ==="
                        # Now that the build stage ran, target/*.jar will exist here!
                        scp -i \$KEY_FILE -o StrictHostKeyChecking=no target/*.jar ${SERVER_USER}@${TARGET_IP}:/home/${SERVER_USER}/
                        
                        echo "=== Step 3: Launching application in background ==="
                        ssh -i \$KEY_FILE -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "nohup java -jar /home/${SERVER_USER}/*.jar > app.log 2>&1 &"
                        
                        echo "=== Deployment Successfully Completed ==="
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo 'CD Direct Deployment - SUCCESS!'
        }
        failure {
            echo 'CD Direct Deployment - FAILED.'
        }
    }
}
