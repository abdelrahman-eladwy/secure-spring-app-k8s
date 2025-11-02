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
        // stage ('Quality Gate'){
        //     steps{
        //         dir('Java-app'){
        //             sh '''
                  
        //             # Get critical and high counts using SpEL query syntax
        //             CRITICAL=$(fcli ssc issue count --av ${APPLICATION_ID}:${VERSION_NAME} -q "cleanName=='Critical'" -o expr="{totalCount}" || echo "0")
        //             HIGH=$(fcli ssc issue count --av ${APPLICATION_ID}:${VERSION_NAME} -q "cleanName=='High'" -o expr="{totalCount}" || echo "0")
                    
        //             # Fail if critical vulnerabilities exist
        //             if [ "$CRITICAL" -gt "0" ]; then
        //                 echo "Quality Gate FAILED: Found $CRITICAL critical vulnerabilities"
        //                 exit 1
        //             fi
        //             if [ "$HIGH" -gt "7" ]; then
        //                 echo "Quality Gate FAILED: Found $HIGH high vulnerabilities (threshold: 7)"
        //                 exit 1
        //             fi
                    
        //             echo "Quality Gate PASSED: Critical=$CRITICAL, High=$HIGH (threshold: 7)"
        //             '''
        //         }
        //     }
        // }
        // stage ('Sonatype SCA Scan'){
        //     steps{
        //         dir('Java-app'){
        //             script {
        //                 // Capture both output and status from Sonatype scan
        //                 def scanOutput = sh(
        //                     script: '''
        //                     java -jar /home/abdelrahman_aeladwy/latest.jar \
        //                         -a admin:123 \
        //                         -i jenkins \
        //                         -s http://localhost:8070 \
        //                         -r sca-results.json \
        //                         ./target/secure-spring-app-1.0.0-SNAPSHOT.jar 2>&1
        //                     ''',
        //                     returnStdout: true
        //                 )
                        
        //                 echo "=== Sonatype Scan Output ==="
        //                 echo scanOutput
                        
        //                 // Parse critical and severe counts from console output
        //                 def criticalMatch = (scanOutput =~ /Number of components affected:\s+(\d+)\s+critical/)
        //                 def severeMatch = (scanOutput =~ /Number of components affected:\s+\d+\s+critical,\s+(\d+)\s+severe/)
                        
        //                 def criticalCount = criticalMatch ? criticalMatch[0][1].toInteger() : 0
        //                 def severeCount = severeMatch ? severeMatch[0][1].toInteger() : 0
                        
        //                 echo "=== Quality Gate Check ==="
        //                 echo "Critical Components: ${criticalCount}"
        //                 echo "Severe Components: ${severeCount}"
                        
        //                 // Fail if critical vulnerabilities found (threshold: 0)
        //                 if (criticalCount > 0) {
        //                     error("Sonatype Quality Gate FAILED: Found ${criticalCount} critical components (threshold: 0)")
        //                 }
                        
        //                 // Optional: Also fail on severe (adjust threshold as needed)
        //                 if (severeCount > 10) {
        //                     error("Sonatype Quality Gate FAILED: Found ${severeCount} severe components (threshold: 10)")
        //                 }
                        
        //                 echo "Sonatype Quality Gate PASSED"
        //             }
        //         }
        //     }
        // }
        stage ('Build Docker Image'){
            steps{
                dir('Java-app'){
                    sh '''
                    docker build -t secure-spring-app:latest .
                    '''
                }
            }
        }
        stage('Trivy Vulnerability Scanner'){
            steps {
                script {
                    // Scan for LOW and MEDIUM severities (informational only)
                    sh '''
                        trivy image secure-spring-app:latest \
                            --severity LOW,MEDIUM \
                            --exit-code 0 \
                            --format json \
                            --output trivy-image-MEDIUM-results.json
                    '''
                    
                    // Scan for HIGH and CRITICAL severities (fail build if found)
                    def scanStatus = sh(
                        script: '''
                            trivy image secure-spring-app:latest \
                                --severity HIGH,CRITICAL \
                                --exit-code 1 \
                                --format json \
                                --output trivy-image-CRITICAL-results.json
                        ''',
                        returnStatus: true
                    )
                    
                    // Parse results to show summary
                    def criticalResults = sh(
                        script: 'cat trivy-image-CRITICAL-results.json | jq -r ".Results[]?.Vulnerabilities // [] | map(select(.Severity == \\"CRITICAL\\")) | length" | awk \'{sum+=$1} END {print sum}\'',
                        returnStdout: true
                    ).trim()
                    
                    def highResults = sh(
                        script: 'cat trivy-image-CRITICAL-results.json | jq -r ".Results[]?.Vulnerabilities // [] | map(select(.Severity == \\"HIGH\\")) | length" | awk \'{sum+=$1} END {print sum}\'',
                        returnStdout: true
                    ).trim()
                    
                    echo "=== Trivy Scan Results ==="
                    echo "Critical vulnerabilities: ${criticalResults}"
                    echo "High vulnerabilities: ${highResults}"
                    
                    if (scanStatus != 0) {
                        error("Trivy Quality Gate FAILED: Found HIGH or CRITICAL vulnerabilities")
                    }
                    
                    echo "Trivy Quality Gate PASSED"
                }
            }
        }
        // stage ('Anchor Grype Scan'){
        //     steps{
        //         dir('Java-app'){
        //             sh '''
        //             grype secure-spring-app:latest --fail-on high
        //             '''
        //         }
        //     }
        // }
    }
}


