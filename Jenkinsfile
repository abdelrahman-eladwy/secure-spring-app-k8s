pipeline{
    agent any
    environment{
        PATH = "/home/abdelrahman_aeladwy/Fortify/OpenText_SAST_Fortify_25.2.0/bin:/usr/local/bin:$PATH"
        GITHUB_URL = "https://github.com/abdelrahman-eladwy/Java-app.git"
        CLIENT_AUTH_TOKEN = credentials('fortify-client-auth-token')
        SSC_USERNAME = 'admin'
        SSC_PASSWORD = credentials('fortify-ssc-password')
        SSC_URL = "http://34.1.41.80:8080/ssc"
        APPLICATION_ID = "jenkins"
        VERSION_NAME = "jenkins"
        SENSOR_VERSION = "25.2"
        GITHUB_TOKEN = credentials('GITHUB_TOKEN')
        USER_EMAIL = "abdoahmed32522@gmail.com"
        SAST_QG_CRITICAL_THRESHOLD = 0
        SAST_QG_HIGH_THRESHOLD = 7
        SONATYPE_QG_CRITICAL_THRESHOLD = 0
        SONATYPE_QG_HIGH_THRESHOLD = 7
        ECR_REGISTRY = "public.ecr.aws/f8a9z5u9/jenkins1" 
        DAST_SETTINGS_CICD_ID = "873edd17-842b-4bef-843a-77764447f1e6"
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
                    echo ${CLIENT_AUTH_TOKEN}
                    fcli ssc session login --client-auth-token=${CLIENT_AUTH_TOKEN} --user=${SSC_USERNAME} --password=${SSC_PASSWORD} --url=${SSC_URL} --insecure
                    scancentral package -o package.zip
                    fcli sc-sast scan start --publish-to=${APPLICATION_ID}:${VERSION_NAME} --sensor-version=${SENSOR_VERSION} --file=package.zip --store=Id
                    fcli sc-sast scan wait-for ::Id:: --interval=30s
                    
                    # Get the latest artifact ID and download the FPR file from SSC
                    ARTIFACT_ID=$(fcli ssc artifact list --av ${APPLICATION_ID}:${VERSION_NAME} -o json | jq -r '.[0].id')
                    echo "Downloading artifact ID: $ARTIFACT_ID"
                    fcli ssc artifact download $ARTIFACT_ID --file sast-results.fpr
                    
                    echo "SAST scan completed and results downloaded to sast-results.fpr"
                    '''
                }
            }
        }
        stage ('Quality Gate'){
            steps{
                dir('Java-app'){
                    sh '''
                  
                    # Get critical and high counts using SpEL query syntax
                    CRITICAL=$(fcli ssc issue count --av ${APPLICATION_ID}:${VERSION_NAME} -q "cleanName=='Critical'" -o expr="{totalCount}" || echo "0")
                    HIGH=$(fcli ssc issue count --av ${APPLICATION_ID}:${VERSION_NAME} -q "cleanName=='High'" -o expr="{totalCount}" || echo "0")
                    
                    # Fail if critical vulnerabilities exist
                    if [ "$CRITICAL" -gt "${SAST_QG_CRITICAL_THRESHOLD}" ]; then
                        echo "Quality Gate FAILED: Found $CRITICAL critical vulnerabilities"
                        exit 1
                    fi
                    if [ "$HIGH" -gt "${SAST_QG_HIGH_THRESHOLD}" ]; then
                        echo "Quality Gate FAILED: Found $HIGH high vulnerabilities (threshold: 7)"
                        exit 1
                    fi
                    
                    echo "Quality Gate PASSED: Critical=$CRITICAL, High=$HIGH (threshold: 7)"
                    '''
                }
            }
        }
        stage ('Sonatype SCA Scan'){
            steps{
                dir('Java-app'){
                    script {
                        // Capture both output and status from Sonatype scan
                        def scanOutput = sh(
                            script: '''
                            java -jar /home/abdelrahman_aeladwy/latest.jar \
                                -a admin:123 \
                                -i jenkins \
                                -s http://localhost:8070 \
                                -r sca-results.json \
                                ./target/secure-spring-app-1.0.0-SNAPSHOT.jar 2>&1
                            ''',
                            returnStdout: true
                        )
                        
                        echo "=== Sonatype Scan Output ==="
                        echo scanOutput
                        
                        // Parse critical and severe counts from console output
                        def criticalMatch = (scanOutput =~ /Number of components affected:\s+(\d+)\s+critical/)
                        def severeMatch = (scanOutput =~ /Number of components affected:\s+\d+\s+critical,\s+(\d+)\s+severe/)
                        
                        def criticalCount = criticalMatch ? criticalMatch[0][1].toInteger() : 0
                        def severeCount = severeMatch ? severeMatch[0][1].toInteger() : 0
                        
                        echo "=== Quality Gate Check ==="
                        echo "Critical Components: ${criticalCount}"
                        echo "Severe Components: ${severeCount}"
                        
                        // Fail if critical vulnerabilities found (threshold: 0)
                        if (criticalCount > ${SONATYPE_QG_CRITICAL_THRESHOLD}) {
                            error("Sonatype Quality Gate FAILED: Found ${criticalCount} critical components (threshold: 0)")
                        }
                        
                        // Optional: Also fail on severe (adjust threshold as needed)
                        if (severeCount > ${SONATYPE_QG_HIGH_THRESHOLD}) {
                            error("Sonatype Quality Gate FAILED: Found ${severeCount} severe components (threshold: 10)")
                        }
                        
                        echo "Sonatype Quality Gate PASSED"
                    }
                }
            }
        }
        stage ('Build Docker Image'){
            steps{
                dir('Java-app'){
                    sh '''
                    docker build -t secure-spring-app:${BUILD_ID} .
                    '''
                }
            }
        }
        stage('Trivy Vulnerability Scanner') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '''
                        echo "=== Scanning image for HIGH vulnerabilities ==="
                        trivy image \
                            --severity HIGH \
                            --exit-code 0 \
                            --no-progress \
                            --format json \
                            --output trivy-image-HIGH-results.json \
                            secure-spring-app:${BUILD_ID}

                        echo "=== Scanning image for CRITICAL vulnerabilities ==="
                        trivy image \
                            --severity CRITICAL \
                            --exit-code 1 \
                            --no-progress \
                            --format json \
                            --output trivy-image-CRITICAL-results.json \
                            secure-spring-app:${BUILD_ID}
                    '''
                }
            }
            post {
                always {
                    sh '''
                        echo "=== Converting Trivy JSON results to HTML & JUnit ==="

                        trivy convert \
                            --format template \
                            --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output trivy-image-HIGH-results.html \
                            trivy-image-HIGH-results.json

                        trivy convert \
                            --format template \
                            --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output trivy-image-CRITICAL-results.html \
                            trivy-image-CRITICAL-results.json

                        trivy convert \
                            --format template \
                            --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output trivy-image-HIGH-results.xml \
                            trivy-image-HIGH-results.json

                        trivy convert \
                            --format template \
                            --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output trivy-image-CRITICAL-results.xml \
                            trivy-image-CRITICAL-results.json
                    '''
                }
            }
        }
        stage('Anchore Grype Scan') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    sh '''
                        echo "=== Running Anchore Grype Scan ==="

                        # Create workspace folder if missing
                        mkdir -p grype-report

                        # JSON report (for archiving)
                        grype secure-spring-app:${BUILD_ID} \
                            --fail-on high \
                            --output json > grype-report/grype-results.json || true

                        # HTML-style text report (for viewing)
                        grype secure-spring-app:${BUILD_ID} \
                            --output table > grype-report/grype-report.html || true
                    '''
                }
            }
            post {
                always {
                    publishHTML([
                        reportDir: 'grype-report',
                        reportFiles: 'grype-report.html',
                        reportName: 'Anchore Grype Report'
                    ])
                    archiveArtifacts artifacts: 'grype-report/**', onlyIfSuccessful: false
                }
            }
        }
        stage('Push To ECR') {
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'jenkins-user', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    script {
                        sh '''
                        # Login to EC
                        aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/f8a9z5u9                         
                        # Tag and push image
                        docker tag secure-spring-app:${BUILD_ID} ${ECR_REGISTRY}:${BUILD_ID}
                        docker push ${ECR_REGISTRY}:${BUILD_ID}
                        '''
                    }
                }
            }
        }
        stage('Change Image Tag') {
            steps {
                sh """
                rm -rf secure-spring-app-k8s
                git clone https://github.com/abdelrahman-eladwy/secure-spring-app-k8s.git
                cd secure-spring-app-k8s
                sed -i 's|image: .*|image: ${ECR_REGISTRY}:${BUILD_ID}|g' deployment.yaml
                cat deployment.yaml
                echo "Image tag changed successfully to BUILD_ID: ${BUILD_ID}"
                git config --global user.email $USER_EMAIL
                git remote set-url origin https://$GITHUB_TOKEN@github.com/abdelrahman-eladwy/secure-spring-app-k8s.git
                git add . 
                git commit -m "FROM CI/CD - Update image tag to ${BUILD_ID}"
                git push origin main
                """
            }
        }
        stage('KubeBench Security Scan') {
            steps {
                dir('secure-spring-app-k8s') {
                    withCredentials([
                        aws(
                            credentialsId: 'jenkins-eks',
                            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                        )
                    ]) {
                        withKubeConfig([credentialsId: 'KUBECONFIG']) {
                            sh '''
                                export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                                export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                                export AWS_DEFAULT_REGION=eu-central-1
                                export AWS_REGION=eu-central-1

                                echo "[INFO] === Running KubeBench Security Scan ==="
                                
                                kubectl get nodes -o wide
                                echo "[SUCCESS] Authenticated to EKS"

                                # Ensure kube-bench namespace exists
                                kubectl create namespace kube-bench --dry-run=client -o yaml | kubectl apply -f -

                                kubectl delete job kube-bench -n kube-bench --ignore-not-found=true
                                sleep 2

                                kubectl apply -f kube-bench-job.yaml -n kube-bench --validate=false

                                echo "Waiting for kube-bench pod to be created..."
                                sleep 10
                                
                                echo "Waiting for kube-bench to finish..."
                                for i in {1..60}; do
                                    POD_STATUS=$(kubectl get pods -n kube-bench -l job-name=kube-bench -o jsonpath='{.items[0].status.phase}' 2>/dev/null || echo "NotFound")
                                    echo "Pod Status: $POD_STATUS"
                                    if [ "$POD_STATUS" = "Succeeded" ] || [ "$POD_STATUS" = "Failed" ]; then
                                        break
                                    fi
                                    sleep 5
                                done

                                echo "[INFO] === KubeBench Results ==="
                                kubectl logs -n kube-bench -l job-name=kube-bench || true

                                echo "[INFO] Cleaning up kube-bench job"
                                kubectl delete job kube-bench -n kube-bench --ignore-not-found=true
                            '''
                        }
                    }
                }
            }
        }
        stage('ScanCentral DAST Scan') {
            steps {
we                echo "[INFO] Logging into SSC at ${SSC_URL}"
                        fcli ssc session login \
                            --user=${SSC_USERNAME} \
                            --password=${SSC_PASSWORD} \
                            --url=${SSC_URL} \
                            --insecure

                        # Create scan name
                        SCAN_NAME="jenkins-dast-scan-${BUILD_ID}"
                        echo "[INFO] Starting DAST scan named: ${SCAN_NAME}"

                        # Start the DAST scan and capture Scan ID
                        fcli sc-dast scan start \
                            --name="${SCAN_NAME}" \
                            --settings=${DAST_SETTINGS_CICD_ID} \
                            --mode=CrawlAndAudit \
                            --store=DastId

                        echo "[INFO] DAST scan started with ID: ::DastId::"

                        # Wait for completion
                        echo "[INFO] Waiting for DAST scan to complete..."
                        fcli sc-dast scan wait-for ::DastId:: --interval=60s --timeout=600s

                        echo "[INFO] DAST scan completed."

                        # Fetch artifact ID generated by this scan
                        echo "[INFO] Retrieving latest DAST artifact ID..."
                        ARTIFACT_ID=$(fcli ssc artifact list --av ${APPLICATION_ID}:${VERSION_NAME} -o json \
                                        | jq -r '[.[] | select(.scanTypes=="WEBINSPECT")] | sort_by(.uploadDate) | reverse | .[0].id'
                                    )

                        echo "[INFO] Found DAST Artifact ID: $ARTIFACT_ID"

                        # Download the DAST FPR file
                        fcli ssc artifact download $ARTIFACT_ID --file dast-results.fpr

                        echo "[INFO] DAST results downloaded: dast-results.fpr"

                        # Show final scan output
                        echo "[INFO] Displaying DAST scan summary:"
                        fcli sc-dast scan get ::DastId:: --output table

                        echo "[INFO] Logging out from SSC"
                        fcli ssc session logout
                    '''
                }
            }
        }
        stage('Upload Security Reports to S3') {
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
                                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY',
                                     credentialsId: 'jenkins-user')]) {
                    sh '''
                        echo "=== Preparing S3 Uploads ==="
                        mkdir -p s3-artifacts

                        # Collect Fortify SAST
                        if [ -f Java-app/sast-results.fpr ]; then
                            cp Java-app/sast-results.fpr s3-artifacts/sast-results-${BUILD_ID}.fpr
                        fi

                        # Collect Fortify DAST
                        if [ -f dast-results.fpr ]; then
                            cp dast-results.fpr s3-artifacts/dast-results-${BUILD_ID}.fpr
                        fi

                        # Collect Trivy Reports (if pipeline enabled)
                        if ls trivy-* >/dev/null 2>&1; then
                            cp trivy-* s3-artifacts/
                        fi

                        # Collect Grype Reports
                        if [ -d grype-report ]; then
                            cp grype-report/\* s3-artifacts/
                        fi

                        # Save KubeBench logs to file
                        echo "=== Saving KubeBench Logs ==="
                        kubectl logs kube-bench > s3-artifacts/kubebench-${BUILD_ID}.log 2>/dev/null || true

                        echo "=== Uploading All Artifacts to S3 ==="
                        aws s3 cp s3-artifacts s3://jenkins-zinad/Pipeline_Build-${BUILD_ID}/ --recursive

                        echo "=== Upload Completed Successfully ==="
                    '''
                }
            }
        }
    }
}


