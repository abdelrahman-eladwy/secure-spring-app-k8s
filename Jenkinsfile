pipeline{
    agent any
    environment{
        PATH = "/usr/local/bin:$PATH"
        GITHUB_URL = "https://github.com/abdelrahman-eladwy/Java-app.git"
    }
    stages{
        stage ('SCM'){
            steps{
                sh '''
                rm -rf Java-app
                git clone ${GITHUB_URL}
                '''
            }
        }
        stage ('BUILD'){
            steps{
                dir('Java-app'){
                    sh 'mvn clean package'
                }
            }
        }
    }
}


