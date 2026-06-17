// Jenkinsfile
def NEXUS_HOST      = "localhost:8083"   
def NEXUS_MAIN_HOST = "localhost:8084"
def MR_IMAGE        = "${NEXUS_HOST}/spring-petclinic"
def MAIN_IMAGE      = "${NEXUS_MAIN_HOST}/spring-petclinic"
def SHORT_COMMIT    = ""

pipeline {
    agent {
        // Runs on any available Jenkins agent
        label "agent"
    }

    // tools {
    //     maven "maven-3.9"
    //     jdk   "jdk-17"
    // }

    environment {
        NEXUS_CREDS = credentials("nexus-credentials")
    }

    stages {

        // ── Shared: always runs first ─────────────────────────────────────
        stage("Prepare") {
            steps {
                script {
                    SHORT_COMMIT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    echo "Short commit: ${SHORT_COMMIT}"
                }
            }
        }

        stage("Verify Java & Maven") {
            steps {
                sh """
                    java -version
                    mvn -v
                """
            }
        }

        // ══ MR PIPELINE ══════════════════════════════════════════════════
        stage("Checkstyle") {
            when {
                changeRequest()   // only runs for merge/pull requests
            }
            steps {
                sh "mvn checkstyle:checkstyle --batch-mode --no-transfer-progress"
            }
            post {
                always {
                    // Publish checkstyle report as job artifact
                    recordIssues(
                        tools: [checkStyle(pattern: "**/checkstyle-result.xml")]
                    )
                    archiveArtifacts artifacts: "**/checkstyle-result.xml",
                                     allowEmptyArchive: true
                }
            }
        }

        stage("Test") {
            when {
                changeRequest()
            }
            steps {
                sh """
                    mvn verify \
                        -Dcheckstyle.skip=true \
                        --batch-mode \
                        --no-transfer-progress
                """
            }
            post {
                always {
                    junit "**/target/surefire-reports/TEST-*.xml"
                }
            }
        }

        stage("Build") {
            when {
                changeRequest()
            }
            steps {
                sh """
                    mvn package \
                        -DskipTests \
                        -Dcheckstyle.skip=true \
                        --batch-mode \
                        --no-transfer-progress
                """
            }
            post {
                success {
                    archiveArtifacts artifacts: "target/*.jar",
                                     fingerprint: true
                }
            }
        }

        stage("Docker Build & Push to MR repo") {
            when {
                changeRequest()
            }
            steps {
                script {
                    docker.withRegistry("http://${NEXUS_HOST}", "nexus-credentials") {
                        def image = docker.build(
                            "${MR_IMAGE}:${SHORT_COMMIT}",
                            "--file Dockerfile ."
                        )
                        image.push()
                        echo "Pushed ${MR_IMAGE}:${SHORT_COMMIT}"
                    }
                }
            }
        }

        // ══ MAIN BRANCH PIPELINE ════════════════════════════════════════
        stage("Docker Build & Push to Main repo") {
            when {
                branch "main"
            }
            steps {
                script {
                    docker.withRegistry("http://${NEXUS_MAIN_HOST}", "nexus-credentials") {
                        def image = docker.build(
                            "${MAIN_IMAGE}:${SHORT_COMMIT}",
                            "--file Dockerfile ."
                        )
                        image.push()
                        // also tag as latest
                        image.push("latest")
                        echo "Pushed ${MAIN_IMAGE}:${SHORT_COMMIT} and :latest"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Pipeline succeeded."
        }
        failure {
            echo "Pipeline failed."
        }
    }
}