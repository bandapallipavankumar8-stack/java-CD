pipeline {
    agent any
    
    environment {
        // 1. Remote machine connection details
        TARGET_IP = '15.207.248.30'
        SERVER_USER = 'ec2-user' 
        
        // 2. Only requires your working target server SSH key credential
        SSH_KEY = credentials('target-server-ssh-key')
    }
    
    stages {
        stage('Deploy Directly to Target Machine') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'target-server-ssh-key', keyFileVariable: 'KEY_FILE')]) {
                    sh """
                        echo "=== Step 1: Stopping old Java applications ==="
                        ssh -i \$KEY_FILE -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "pkill -f '\\.jar'" || true
                        sleep 2
                        
                        echo "=== Step 2: Uploading JAR directly from Jenkins to Target ==="
                        # This looks inside your current Jenkins build workspace folder for any .jar file
                        # and copies it safely over the SSH bridge to the ec2-user home folder
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
