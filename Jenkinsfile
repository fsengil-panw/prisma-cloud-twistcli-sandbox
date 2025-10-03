pipeline {
    // Request a dynamic agent from the EC2 Spot Instance template you configured
    agent {
        label 'ec2-spot-sandbox'
    }

    environment {
        // Securely load the Prisma Cloud URL from Jenkins credentials
        PRISMA_URL = credentials('prisma-console-url')
    }

    stages {
        stage('Setup Analysis Environment') {
            steps {
                echo "Agent is running on a temporary EC2 Spot Instance."
                echo "Downloading twistcli from ${PRISMA_URL}..."

                // Use the Prisma Cloud credentials to download the twistcli tool
                withCredentials([usernamePassword(credentialsId: 'prisma-cloud-credentials', usernameVariable: 'PRISMA_USER', passwordVariable: 'PRISMA_PASS')]) {
                    sh """
                        curl -k -u ${PRISMA_USER}:${PRISMA_PASS} \
                             --output twistcli ${PRISMA_URL}/api/v1/util/twistcli
                        chmod a+x twistcli
                    """
                }
            }
        }

        stage('Run Prisma Cloud Sandbox Analysis') {
            steps {
                // Ensure a placeholder malware sample exists in your repo for testing
                // In a real scenario, this would be the file you want to analyze
                touch 'malware_sample.exe'

                echo "Submitting malware_sample.exe to Prisma Cloud Sandbox..."
                withCredentials([usernamePassword(credentialsId: 'prisma-cloud-credentials', usernameVariable: 'PRISMA_USER', passwordVariable: 'PRISMA_PASS')]) {
                    // Execute sandbox scan. --output-file saves the JSON result.
                    // The 'script' block allows us to check the exit code.
                    script {
                        def scanResult = sh(
                            script: """
                                ./twistcli sandbox \
                                  --address ${PRISMA_URL} \
                                  --user ${PRISMA_USER} \
                                  --password ${PRISMA_PASS} \
                                  --output-file prisma_report.json \
                                  malware_sample.exe
                            """,
                            returnStatus: true
                        )

                        // twistcli exits with a non-zero status code if it finds a vulnerability or malware
                        if (scanResult != 0) {
                            echo "Scan complete. Prisma Cloud detected potential issues."
                            // We will inspect the JSON report in the next stage
                        } else {
                            echo "Scan complete. No issues detected by exit code."
                        }
                    }
                }
            }
        }

        stage('Process and Archive Results') {
            steps {
                script {
                    // readJSON is a built-in pipeline step to parse JSON files
                    def report = readJSON file: 'prisma_report.json'
                    def verdict = report.results[0].malware.verdict

                    echo "Prisma Cloud Sandbox Verdict: ${verdict}"

                    archiveArtifacts artifacts: 'prisma_report.json', followSymlinks: false

                    // Fail the pipeline if the verdict is definitively 'malware'
                    if (verdict == 'malware') {
                        error("PIPELINE FAILED: Prisma Cloud identified the sample as malware.")
                    }
                }
            }
        }
    }
    
    // No 'post' block for teardown is needed!
    // The Jenkins EC2 plugin automatically terminates the agent (the Spot Instance)
    // as soon as the pipeline job is finished.
}