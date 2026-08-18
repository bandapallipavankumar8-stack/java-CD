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
                    // Merged into one single block with double quotes
                    sh """
                        ssh -v -o ConnectTimeout=10 -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "
                            pkill -f '.jar' || true
                            aws s3 cp ${S3_BUCKET} . --recursive --exclude '*' --include '*.jar'
                            nohup java -jar *.jar > app.log 2>&1 &
                        "
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
