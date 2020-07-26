// This pipeline performs a full suite of automated security testing
// - lint: Lint Docker image
// - detect new secrets: Detect new secrets
// - sonarscanner: Scan source code
// - build: Build Docker image
// - push: Push Docker image to registry
// - dependency check: Test for insecure third party libraries
// - sidecar: Run sidecar
// - scan: Scan container image
// - nikto: Perform a Nikto scan

pipeline {
    environment { // Environment variables defined for all steps
        DOCKER_IMAGE = "registry.demo.local:5000/juice-shop"
        TOOLS_IMAGE = "registry.demo.local:5000/tools-image"
        JENKINS_UID = 1001 // User ID under which Jenkins runs
        JENKINS_GID = 900 // Group ID under which Jenkins runs
        SONAR_KEY = "juice-shop"
    }

    agent any

    stages {
        stage("lint") {
            agent {
                docker {
                    image "docker.io/hadolint/hadolint:v1.18.0"
                    reuseNode true
                }
            }
            steps {
                sh label: "Lint Dockerfile", script: "hadolint Dockerfile > hadolint-results.txt"
            }
        }

        // Detect new secrets added since last successful build
        stage("detect new secrets") {
            agent {
                docker {
                    image "${TOOLS_IMAGE}"
                    // Make sure that username can be mapped correctly
                    args "--volume /etc/passwd:/etc/passwd:ro"
                    reuseNode true
                }
            }
            steps {
                // Determine commit of previous successful build when this is master
                script {
                    def result = sh label: "detect-secrets",
                        script: """\
                            detect-secrets-hook --no-verify \
                                                -v \
                                                --baseline .secrets.baseline.json \
                            \$(git diff-tree --no-commit-id --name-only -r ${GIT_COMMIT} | xargs -n1)
                        """,
                        returnStatus: true
                    // Exit code 1 is generated when secrets are detected or no baseline is present
                    // Exit code 3 is generated only when .secrets.baseline.json is updated,
                    // eg. when the line numbers don't match anymore
                    if (result == 1) {
                        // There are (unaudited) secrets detected: fail stage
                        unstable(message: "unaudited secrets have been found")
                    }
                }
            }
        }

        stage("sonarscanner") {
            agent {
                docker {
                    image "${TOOLS_IMAGE}"
                    // Make sure that username can be mapped correctly
                    args "--volume /etc/passwd:/etc/passwd:ro --network lab"
                    reuseNode true
                }
            }
            steps {
                withSonarQubeEnv("sonarqube.demo.local") {
                    sh label: "install prerequisites",
                        script: "npm install -D typescript"
                    sh label: "sonar-scanner",
                        script: """\
                            sonar-scanner \
                            '-Dsonar.buildString=${BRANCH_NAME}-${BUILD_ID}' \
                            '-Dsonar.projectKey=${SONAR_KEY}' \
                            '-Dsonar.projectVersion=${BUILD_ID}' \
                            '-Dsonar.sources=${WORKSPACE}'
                        """
                }
            }
        }

        stage("Dependency-Check") {
            agent {
                docker {
                    image "owasp/dependency-check:5.3.0"
                    // Set user to root, as the container runs by default under 100:101
                    args '''\
                        --user 0 \
                        --volume dependency-check:/usr/share/dependency-check/data:rw \
                        --volume ${WORKSPACE}:/src:ro \
                        --volume ${WORKSPACE}/reports:/reports:rw \
                        --entrypoint ""
                    '''
                    reuseNode true
                }
            }
            steps {
                script {
                    // Fail stage when a vulnerability having a base CVSS score of 6 or higher is found
                    def result = sh label: "dependency-check", returnStatus: true,
                        script: """\
                            mkdir -p reports &>/dev/null
                            # Fix permissions as this container is being run as root
                            chown "${JENKINS_UID}:${JENKINS_GID}" reports
                            /usr/share/dependency-check/bin/dependency-check.sh \
                            --failOnCVSS 6 \
                            --out "${WORKSPACE}/reports" \
                            --project "${JOB_BASE_NAME}" \
                            --scan "/src"
                            # Fix permissions as this container is being run as root
                            chown "${JENKINS_UID}:${JENKINS_GID}" reports/dependency-check-report.html
                        """
                    if (result > 0) {
                        unstable(message: "Insecure libraries found")
                    }
                }
            }
        }

        stage("Build image") {
            steps {
                script {
                    // Use commit tag if it has been tagged
                    tag = sh(returnStdout: true, script: "git tag --contains").trim()
                    if ("$tag" == "") {
                        if ("${BRANCH_NAME}" == "master") {
                            tag = "latest"
                        } else {
                            tag = "${BRANCH_NAME}"
                        }
                    }
                    image = docker.build("$DOCKER_IMAGE:$tag")
                }
            }
        }

        stage("Push to registry") {
            steps {
                script {
                    sh label: "Push to registry", script: "docker push ${DOCKER_IMAGE}:$tag"
                }
            }
        }

        stage("Launch sidecar") {
            steps {
                sh label: "Start sidecar container",
                    script: """\
                        docker run --detach \
                                   --network lab \
                                   --name ${JOB_BASE_NAME}-${BUILD_ID} \
                                   --rm \
                                   ${DOCKER_IMAGE}:${tag}
                    """
            }
        }

        stage("Scan container") {
            agent {
                docker {
                    image "$TOOLS_IMAGE"
                    // Make sure that the container can access anchore-engine_api_1
                    args "--network=lab"
                    reuseNode true
                }
            }
            steps {
                // Continue the build, even after policy failure
                script {
                    sh label: "Ensure anchore is available",
                        script: "anchore-cli system status"
                    sh label: "Add to queue",
                        script: "anchore-cli image add ${DOCKER_IMAGE}:$tag"
                    sh label: "Wait for analysis",
                        script: "anchore-cli image wait ${DOCKER_IMAGE}:$tag"
                    sh label: "Generate list of vulnerabilities",
                        script: "anchore-cli image vuln $DOCKER_IMAGE:$tag all | tee anchore-results.txt"
                    def result = sh label: "Check policy",
                        script: "anchore-cli evaluate check ${DOCKER_IMAGE}:$tag --detail >> anchore-results.txt"
                    if (result > 0) {
                        unstable(message: "Policy check failed")
                    }
                }
            }
        }

        stage("nikto") {
            agent {
                docker {
                    image "$TOOLS_IMAGE"
                    // Make sure that the container can access the sidecar
                    args "--network=lab"
                    reuseNode true
                }
            }
            steps {
                script {
                    // Fail stage when Nikto finds something
                    def result = sh label: "nikto", returnStatus: true,
                        script: """\
                            mkdir -p reports &>/dev/null
                            curl --max-time 120 \
                                --retry 60 \
                                --retry-connrefused \
                                --retry-delay 5 \
                                --fail \
                                --silent http://${JOB_BASE_NAME}-${BUILD_ID}:3000 || exit 1
                            rm reports/nikto.html &> /dev/null
                            nikto.pl -ask no \
                                -nointeractive \
                                -output reports/nikto.html \
                                -Plugins '@@ALL;-sitefiles' \
                                -Tuning x7 \
                                -host http://${JOB_BASE_NAME}-${BUILD_ID}:3000 > nikto.pl-results.txt
                        """
                    if (result > 0) {
                        unstable(message: "Web server scanner issues found")
                    }
                }
            }
        }

        stage("OWASP ZAP") {
            agent {
                docker {
                    image "owasp/zap2docker-weekly"
                    // Make sure that the container can access the sidecar
                    args "--network=lab --tty --volume ${WORKSPACE}:/zap/wrk"
                    reuseNode true
                }
            }
            steps {
                script {
                    def result = sh label: "OWASP ZAP", returnStatus: true,
                        script: """\
                            mkdir -p reports &>/dev/null
                            curl --max-time 120 \
                                --retry 60 \
                                --retry-connrefused \
                                --retry-delay 5 \
                                --fail \
                                --silent http://${JOB_BASE_NAME}-${BUILD_ID}:3000 || exit 1
                            zap-baseline.py \
                            -m 5 \
                            -T 5\
                            -I \
                            -r reports/zapreport.html \
                            -t "http://${JOB_BASE_NAME}-${BUILD_ID}:3000"
                    """
                    if (result > 0) {
                        unstable(message: "OWASP ZAP issues found")
                    }
                }
            }
        }
    }

    post {
        always {
            sh label: "Stop sidecar container", script: "docker stop ${JOB_BASE_NAME}-${BUILD_ID}"
            archiveArtifacts artifacts: "*-results.txt"
            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: false,
                reportDir: "reports",
                reportFiles: "dependency-check-report.html",
                reportName: "Dependency Check Report"
            ])
            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: "reports",
                reportFiles: "nikto.html",
                reportName: "Nikto.pl scanreport"
            ])
            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: "reports",
                reportFiles: "zapreport.html",
                reportName: "OWASP ZAP scanreport"
            ])
        }
    }
}
