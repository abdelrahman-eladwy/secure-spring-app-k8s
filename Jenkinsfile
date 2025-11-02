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
        stage ('Sonatype SCA Scan'){
            steps{
                dir('Java-app'){
                    script {
                        def scanStatus = sh(
                            script: '''
                            java -jar /home/abdelrahman_aeladwy/latest.jar \
                                -a admin:123 \
                                -i jenkins \
                                -s http://localhost:8070 \
                                -r sca-results.json \
                                ./target/secure-spring-app-1.0.0-SNAPSHOT.jar
                            ''',
                            returnStatus: true
                        )
                        def scanOutput = sh(
                            script: '''
                            if [ -f sca-results.json ]; then
                                echo "=== Sonatype Scan Results ==="
                                echo "Full JSON structure:"
                                cat sca-results.json | jq '.'
                                echo ""
                                echo "Policy Action: $(jq -r '.policyAction' sca-results.json)"
                                
                                # Try different JSON paths to find component counts
                                CRITICAL=$(jq -r '.componentsAffected.critical // .components.critical // 0' sca-results.json)
                                SEVERE=$(jq -r '.componentsAffected.severe // .components.severe // 0' sca-results.json)
                                
                                echo "Critical Components: $CRITICAL"
                                echo "Severe Components: $SEVERE"
                                echo "$CRITICAL"
                            else
                                echo "0"
                            fi
                            ''',
                            returnStdout: true
                        ).trim()
                        
                        def criticalCount = scanOutput.split('\n')[-1].toInteger()
                        
                        echo "Critical components found: ${criticalCount}"
                        
                        // Fail if critical vulnerabilities found
                        if (criticalCount > 0) {
                            error("Sonatype Quality Gate FAILED: Found ${criticalCount} critical components (threshold: 0)")
                        }
                        
                        // Also check exit code
                        if (scanStatus == 1) {
                            error("Sonatype Quality Gate FAILED: Policy violations detected (exit code: ${scanStatus})")
                        }
                        
                        echo "Sonatype Quality Gate PASSED: No critical vulnerabilities"
                    }
                }
            }
        }
    }
}


