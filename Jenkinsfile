pipeline {
    agent any
    
    environment {
        S3_BUCKET = 's3://code-version/packages/'
        TARGET_IP = '15.207.248.30'
        // Hardcoded target user configuration
        SERVER_USER = 'ec2-user' 
        SSH_KEY = credentials('target-server-ssh-key')
    }
    
    stages {
        stage('Deploy to Target Machine') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'target-server-ssh-key', keyFileVariable: 'KEY_FILE')]) {
                    sh """
                        # 1. Stop any old instances of the jar running
                        ssh -i \$KEY_FILE -o StrictHostKeyChecking=no ec2-user@${TARGET_IP} "pkill -f .jar || true"
                        
                        # 2. Download the package from S3 onto the target machine
                        ssh -i \$KEY_FILE -o StrictHostKeyChecking=no ec2-user@${TARGET_IP} "aws s3 cp ${S3_BUCKET} . --recursive --exclude '*' --include '*.jar'"
                        
                        # 3. Launch the new package in the background
                        ssh -i \$KEY_FILE -o StrictHostKeyChecking=no ec2-user@${TARGET_IP} "nohup java -jar *.jar > app.log 2>&1 &"
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
