pipeline{
    agent any
    environment{
        PATH = "/home/abdelrahman_aeladwy/Fortify/OpenText_SAST_Fortify_25.2.0/bin:/usr/local/bin:$PATH"
        GITHUB_URL = "https://github.com/abdelrahman-eladwy/Java-app.git"
        CLIENT_AUTH_TOKEN = 'CHANGEME321!'
        SSC_USERNAME = 'admin'
        SSC_PASSWORD = 'Jpia331##'
        SSC_URL = "http://34.1.41.80:8080/ssc"
        APPLICATION_ID = "jenkins"
        VERSION_NAME = "jenkins"
        SENSOR_VERSION = "25.2"
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
        // stage ('Fortify SAST Scan'){
        //     steps{
        //         dir('Java-app'){
        //             sh '''
        //             echo ${CLIENT_AUTH_TOKEN}
        //             fcli ssc session login --client-auth-token=${CLIENT_AUTH_TOKEN} --user=${SSC_USERNAME} --password=${SSC_PASSWORD} --url=${SSC_URL} --insecure
        //             scancentral package -o package.zip
        //             fcli sc-sast scan start --publish-to=${APPLICATION_ID}:${VERSION_NAME} --sensor-version=${SENSOR_VERSION} --file=package.zip --store=Id
        //             fcli sc-sast scan wait-for ::Id:: --interval=30s
        //             '''
        //         }
        //     }
        // }
        stage ('Quality Gate'){
            steps{
                dir('Java-app'){
                    sh '''
                    # Check for critical and high severity vulnerabilities
                    echo "Checking vulnerability counts..."
                    fcli ssc issue count --av ${APPLICATION_ID}:${VERSION_NAME}
                    
                    # Get critical count
                    echo $(fcli ssc issue count --av ${APPLICATION_ID}:${VERSION_NAME})
                    CRITICAL=$(fcli ssc issue count --av ${APPLICATION_ID}:${VERSION_NAME} -q "friority:Critical" -o expr="{totalCount}" || echo "0")
                    HIGH=$(fcli ssc issue count --av ${APPLICATION_ID}:${VERSION_NAME} -q "friority:High" -o expr="{totalCount}" || echo "0")
                    
                    echo "Critical vulnerabilities: $CRITICAL"
                    echo "High vulnerabilities: $HIGH"
                    
                    # Fail if critical vulnerabilities exist
                    if [ "$CRITICAL" -gt "0" ]; then
                        echo "Quality Gate FAILED: Found $CRITICAL critical vulnerabilities"
                        exit 1
                    fi
                    if [ "$HIGH" -gt "7" ]; then
                        echo "Quality Gate FAILED: Found $HIGH high vulnerabilities"
                        exit 1
                    fi
                    
                    echo "Quality Gate PASSED"
                    '''
                }
            }
        }
    }
}


