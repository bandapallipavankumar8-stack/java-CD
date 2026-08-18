pipeline { 
    agent any 
    
    environment {
        S3_BUCKET = 's3://code-version/packages/'
        TARGET_IP = '15.207.248.30'
        // Name of your SSH credentials ID stored safely in Jenkins
        SSH_CRED_ID = 'target-server-ssh-key' 
        // The user on your target machine (e.g., ec2-user, ubuntu, root)
        SERVER_USER = 'ec2-user' 
    }
    
    tools { 
        jdk 'JDK-17' 
        maven 'Maven-3.9' 
    } 
    
    stages { 
        stage('Checkout') { 
            steps { 
                checkout scm 
            } 
        } 
        stage('Build') { 
            steps { 
                sh 'mvn clean package' 
            } 
        } 
        stage('Test') { 
            steps { 
                sh 'mvn test' 
            } 
        } 
        stage('Archive') { 
            steps { 
                archiveArtifacts( artifacts: 'target/*.jar', fingerprint: true ) 
            } 
        }
        stage('Upload to S3') {
            steps {
                sh "aws s3 cp target/ ${S3_BUCKET} --recursive --exclude '*' --include '*.jar'"
            }
        }
        stage('Deploy') {
            steps {
                // sshagent handles secure SSH keys without exposing passwords
                sshagent([SSH_CRED_ID]) {
                    sh """
                        # 1. Stop the old running application first (if applicable)
                        ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "pkill -f 'target/*.jar' || true"
                        
                        # 2. Command the target machine to pull the latest JAR from S3
                        ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "aws s3 cp ${S3_BUCKET} . --recursive --exclude '*' --include '*.jar'"
                        
                        # 3. Start the application in the background
                        ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "nohup java -jar *.jar > app.log 2>&1 &"
                    """
                }
            }
        }
    } 
    post { 
        success { 
            echo 'CI/CD SUCCESS - JAR uploaded to S3 and deployed to target machine!' 
        } 
        failure { 
            echo 'CI/CD FAILED' 
        } 
    } 
}
