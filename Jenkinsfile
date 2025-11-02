pipeline{
    agent any
    environment{
        PATH = "/usr/local/bin:$PATH"
        GITHUB_URL = "https://github.com/abdelrahman-eladwy/Java-app.git"
        CLIENT_AUTH_TOKEN = 'CHANGME321!'
        SSC_USERNAME = 'admin'
        SSC_PASSWORD = 'Jpia331##'
        SSC_URL = "http://34.1.41.80:8080/ssc"
        APPLICATION_ID = "jenkins"
        SENSOR_VERSION = "jenkins"
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
        stage ('Build Application'){
            steps{
                dir('Java-app'){
                    sh 'mvn clean package'
                }
            }
        }
        stage ('Fortify SAST Scan'){
            steps{
                dir('Java-app'){
                    sh '''
                    fcli ssc session login --client-auth-token=${CLIENT_AUTH_TOKEN} --user=${SSC_USERNAME} --password=${SSC_PASSWORD} --url=${SSC_URL} --insecure
                    scancentral package -o package.zip
                    fcli sc-sast scan start --publish-to=${APPLICATION_ID} --sensor-version=${SENSOR_VERSION} --file=package.zip --store=Id
                    fcli sc-sast scan wait-for ::Id:: --interval=30s

                    '''
                }
            }
        }
    }
}


