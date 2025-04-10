pipeline {
    agent any

    stages {
        stage('Detect Changes') {
            steps {
                script {
                    def affectedServices = []
                    def changedFiles = sh(script: 'git diff --name-only HEAD~1', returnStdout: true).trim().split("\n")
                    echo "Changed files: ${changedFiles}"

                    for (file in changedFiles) {
                        if (file.startsWith("spring-petclinic-") && file.split("/").size() > 1) {
                            def service = file.split("/")[0]
                            if (!affectedServices.contains(service)) {
                                affectedServices << service
                            }
                        }
                    }

                    if (affectedServices.isEmpty()) {
                        echo "No relevant service changes detected. Skipping pipeline."
                        currentBuild.result = 'SUCCESS'
                        return
                    }

                    echo "Affected services: ${affectedServices}"
                    env.AFFECTED_SERVICES = affectedServices.join(',')
                }
            }
        }

        stage('Test and Coverage') {
            when {
                expression { return env.AFFECTED_SERVICES != null && env.AFFECTED_SERVICES != "" }
            }
            steps {
                script {
                    def affectedServices = env.AFFECTED_SERVICES.split(',')
                    sh 'mkdir -p target'
                    def jacocoExecFiles = []
                    for (service in affectedServices) {
                        echo "Testing service: ${service} on ${env.NODE_NAME}"
                        dir(service) {
                            timeout(time: 10, unit: 'MINUTES') {
                                retry(3) {
                                    sh 'mvn clean test -B'
                                }
                            }
                            sh 'mvn jacoco:report -B'
                            jacocoExecFiles << "${service}/target/jacoco.exec"
                        }
                    }

                    if (jacocoExecFiles) {
                        echo "Merging JaCoCo exec files: ${jacocoExecFiles}"
                        def mergeXmlTemplate = readFile 'jacoco-merge.xml'
                        def jacocoExecIncludes = jacocoExecFiles.collect { "<include>${it}</include>" }.join('\n                                        ')
                        def mergeXmlContent = mergeXmlTemplate.replace('${JACOCO_EXEC_FILES}', jacocoExecIncludes)
                        writeFile file: 'jacoco-merge-updated.xml', text: mergeXmlContent
                        sh 'mvn -f jacoco-merge-updated.xml verify -B'
                    }
                }
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'

                    script {
                        if (env.AFFECTED_SERVICES != null && env.AFFECTED_SERVICES != "") {
                            def affectedServices = env.AFFECTED_SERVICES.split(',')
                            echo "Generating aggregated JaCoCo report for all services"
                            jacoco(
                                execPattern: 'target/jacoco-aggregated.exec',
                                classPattern: affectedServices.collect { "${it}/target/classes" }.join(','),
                                sourcePattern: affectedServices.collect { "${it}/src/main/java" }.join(','),
                                exclusionPattern: affectedServices.collect { "${it}/src/test/**" }.join(',')
                            )
                        }
                    }
                }
            }
        }

        stage('Build') {
            when {
                expression { return env.AFFECTED_SERVICES != null && env.AFFECTED_SERVICES != "" }
            }
            steps {
                script {
                    def affectedServices = env.AFFECTED_SERVICES.split(',')
                    for (service in affectedServices) {
                        echo "Building service: ${service} on ${env.NODE_NAME}"
                        dir(service) {
                            sh 'mvn clean package -DskipTests -am -q -B'
                        }
                    }
                }
            }
        }
    }
}
