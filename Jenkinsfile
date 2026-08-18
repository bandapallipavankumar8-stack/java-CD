pipeline {
    agent any
    
    environment {
        TARGET_IP = '15.207.248.30'
        SERVER_USER = 'ec2-user' 
        SSH_KEY = credentials('target-server-ssh-key')
    }
    
    stages {
        stage('Build & Package Code') {
            steps {
                // Changed from 'mvn' to using the local folder's execution wrapper
                sh 'chmod +x mvnw || true'
                sh './mvnw clean package -DskipTests'
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
