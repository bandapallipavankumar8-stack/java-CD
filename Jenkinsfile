pipeline {
    agent any
    
    environment {
        // 1. Storage bucket path where your package is located
        S3_BUCKET = 's3://code-version/packages/'
        
        // 2. Production target machine details
        TARGET_IP = '15.207.248.30'
        SERVER_USER = 'ec2-user' 
        
        // 3. The ID of your SSH private key stored in Jenkins credentials
        SSH_CRED_ID = 'target-server-ssh-key' 
    }
    
    stages {
        stage('Deploy to Target Machine') {
            steps {
                // The sshagent plugin handles secure authentication using your SSH key
                sshagent([SSH_CRED_ID]) {
                    sh """
                        # Stop any older copy of the Java program currently running on the server
                        ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "pkill -f '\\.jar' || true"
                        
                        # Command the target server to pull down the package from S3
                        ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "aws s3 cp ${S3_BUCKET} . --recursive --exclude '*' --include '*.jar'"
                        
                        # Launch the JAR file in the background and record logs to app.log
                        ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "nohup java -jar *.jar > app.log 2>&1 &"
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo 'CD SUCCESS - Package pulled from S3 and deployed successfully!'
        }
        failure {
            echo 'CD FAILED - Deployment failed.'
        }
    }
}
