pipeline { 
    agent any 
    
    environment {
        S3_BUCKET = 's3://code-version/packages/'
        TARGET_IP = '15.207.248.30'
        SSH_CRED_ID = 'target-server-ssh-key' 
        SERVER_USER = 'ec2-user' 
    }
    
    tools { 
        // Changed to 'jdk' and 'Maven' to match your system configurations
        jdk 'jdk' 
        maven 'Maven' 
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
                sshagent([SSH_CRED_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "pkill -f 'target/*.jar' || true"
                        ssh -o StrictHostKeyChecking=no ${SERVER_USER}@${TARGET_IP} "aws s3 cp ${S3_BUCKET} . --recursive --exclude '*' --include '*.jar'"
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
