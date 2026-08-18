pipeline {
    agent any
    
    environment {
        S3_BUCKET = 's3://code-version/packages/'
        TARGET_IP = '15.207.248.30'
        SERVER_USER = 'ec2-user' 
        SSH_KEY = credentials('target-server-ssh-key')
        
        // This maps the code variables to the Jenkins IDs you just created
        AWS_ACCESS_KEY_ID = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
    }
    
    stages {
        stage('Deploy to Target Machine') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'target-server-ssh-key', keyFileVariable: 'KEY_FILE')]) {
                    sh """
                        # 1. Stop any old instance cleanly
                        ssh -i \$KEY_FILE -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "pkill -f '\\.jar'" || true
                        sleep 2
                        
                        # 2. Download your package from S3 using the new credentials
                        ssh -i \$KEY_FILE -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "AWS_ACCESS_KEY_ID='${AWS_ACCESS_KEY_ID}' AWS_SECRET_ACCESS_KEY='${AWS_SECRET_ACCESS_KEY}' aws s3 cp ${S3_BUCKET} . --recursive --exclude '*' --include '*.jar'"
                        
                        # 3. Launch your application in the background
                        ssh -i \$KEY_FILE -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "nohup java -jar *.jar > app.log 2>&1 &"
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
