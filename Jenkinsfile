pipeline {
    agent any
    
    environment {
        S3_BUCKET = 's3://code-version/packages/'
        TARGET_IP = '15.207.248.30'
        SERVER_USER = 'ec2-user' 
        SSH_CRED_ID = 'target-server-ssh-key' 
    }
    
    stages {
        stage('Deploy to Target Machine') {
            steps {
                sshagent([SSH_CRED_ID]) {
                    // Separate commands with simple quotes to avoid escape failures
                    
                    // 1. Stop any old instance of your app running on the target machine
                    sh "ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} 'pkill -f .jar || true'"
                    
                    // 2. Download the package from S3 onto the target machine
                    sh "ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} 'aws s3 cp ${S3_BUCKET} . --recursive --exclude \"*\" --include \"*.jar\"'"
                    
                    // 3. Launch the new package in the background
                    sh "ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} 'nohup java -jar *.jar > app.log 2>&1 &'"
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
