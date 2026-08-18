pipeline {
    agent any
    
    environment {
        S3_BUCKET = 's3://code-version/packages/'
        TARGET_IP = '15.207.248.30'
        SERVER_USER = 'ec2-user' 
        SSH_KEY = credentials('-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAtFsb4tolPpCtiZLa3U+UMz5EyElYsUlaelbNdzYZBIqbl+R8
fWVNNqlUvxeSyDLllxgq6tqscE4gSFn4y+SzMC1M3x/tncQMMh6buNmoonOWCkjG
Jj1N0PMctWgg0QwHUiTqymIovPvRF3Sxr8DJZ4DtgTLOYZg+3Ub9EsR5km9UciBl
3B5CfeQg2e/vNJPv8WICCJC1CpV5pc78TF8VSuOSE3/D/nZpxs0c1M3KO+ewClA7
kjrChglBZdMSLH2GsDcEPoZHKAGdW8oXRHh5XcVcBVevF2Fz7W4mFO6+kyeqxB42
8K+c75gCZ0kBUZ9KyWGjO3xblM50bndkQJnbQQIDAQABAoIBAFLW/koGVNEV9v+X
unuoj7OxyDoOpnRX8vz6XcmByZ/yYmE0C5I1M3AF+u0C0OKvrhDmgt/4rPewitdw
q/xLAZsBU6uwqJ2sbMWglXokT3a+jI4QuyLZSaLN58PTHi+mzL4IQufOilOzfmi9
qtfFPz0RVQXg5jahjU5pytgR8p01S6DEDbjWQEGpNmsNzmR711+t1DutVU7s1xPC
M1jRElqdLzpNkAzxi6Ks3sLni5BKNjBtxDp8wrd72zQ7AMMg8NmIrGnlnmnML0I3
lpD4IW2cTGq2T9uapXRLXh5L9KeIP20ijifWDi5roCaOtps5rscmtWBWy+qLLDsb
b+jBvtkCgYEA6EPgI1nMGK+liYS0L6poyqWRttWxSkQfjtQ8CDmfSTHyP1HLTXAO
ZGyLLCIRNz5zhdBJZo2OnyHsLnXre7TlM/FZfiwMcN9t9YQ+H5Ba+vsnNRrTeWTI
1N4McJUao4xdxqmH4Pp/1IURqKZjWlogAcZpvloDCn3AMkBYcTuBIpcCgYEAxslF
ss+TR+UgOs13ifn3M7CQUwKN5wFt0i9MgqqrJ9QpsSBEcJ2qrHIMc8Ur7VNMnYYs
9dfYgQomIkLn777UnqbnaANAs6Q3HlsrwkqEt4mAlyYXmn4i+q2+dGRxlRG8Elsr
gmKqzsGg8pOLTvJS9cwaKZIO4+p7uIZ3tbOPI+cCgYEAgzFjt1QPfpooLMcyaAIf
cueWqOmHXOWh1bF3v0Wc/WEi7jUrWrBC0OKmseUESGoUIq+F5lFrD+O/Xnbo7lU9
aduXqzcCR/dMSvPJi1akrUOT3+EpNlaBQguyhx0RkPPGPGKiB6g28DnBwbtKP0zM
63PBYu3A7fodx8SksEDmLj0CgYBwCXaP/jALQFc27SDnkgvChUwCjRj/Tq3f3aqo
pppKm2hYHVCVjDdqc+kSwtksLFutGLd0ZA/xQpAVlVH1rL9XH8iitdqcpPwvzsDO
A4Pjkcr45Y4+E8ORN6V1IjtmAhXW3q2aEhQk7brRnVjRyP/66ur/7QMZb8oFSTxl
G2uclwKBgDJJxJThh4M+JaC2A1wEjBUpqPAX6lUvjN0oIjGNOyBO456z9ErtRCy+
YNpvd2sBWycmeyTwcMlVc4Lt2ZR91aw9uiDO6jhz1hq3YL/Ly8FMZdGSF8mxBmaV
1P64Aon6UWre6JuxPSbslYCSM2jEnErtxwOyh305VjnkinJMcwSk
-----END RSA PRIVATE KEY-----)
    }
    
    stages {
        stage('Deploy to Target Machine') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'target-server-ssh-key', keyFileVariable: 'KEY_FILE')]) {
                    sh """
                        # 1. Stop any old instance (Using an isolated execution style so it never drops the session)
                        ssh -i \$KEY_FILE -o StrictHostKeyChecking=no ec2-user@${TARGET_IP} "pkill -f '\\.jar'" || true
                        sleep 2
                        
                        # 2. Download the package from S3 onto the target machine
                        ssh -i \$KEY_FILE -o StrictHostKeyChecking=no ec2-user@${TARGET_IP} "aws s3 cp ${S3_BUCKET} . --recursive --exclude '*' --include '*.jar'"
                        
                        # 3. Launch the new package in the background silently
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
