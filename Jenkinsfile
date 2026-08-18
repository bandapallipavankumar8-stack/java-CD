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
                    // Separate commands to prevent session drops from breaking the whole block
                    
                    // 1. Stop old application (using a cleaner pkill string)
                    sh "ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} 'pkill -f .jar || true'"
                    
                    // 2. Download newest package from S3
                    sh "ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} 'aws s3 cp ${S3_BUCKET} . --recursive --exclude \"*\" --include \"*.jar\"'"
                    
                    // 3. Launch application silently in the background
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
