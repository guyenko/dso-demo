pipeline {
    agent {
        kubernetes {
            yamlFile 'build-agent.yaml'
            defaultContainer 'maven'
            idleMinutes 1
        }
    }

    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn compile'
                }
            }
        }

        stage('Static Analysis') {
            parallel { // Start parallel block here
                stage('SCA') {
                    steps {
                        container('maven') {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh 'mvn org.owasp:dependency-check-maven:check'
                            }
                        }
                    }
                    post {
                        always {
                            archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: true
                        }
                    }
                }

                stage('Generate SBOM') {
                    steps {
                        container('maven') {
                            sh 'mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom'
                        }
                    }
                    post {
                        success {
                            archiveArtifacts allowEmptyArchive: true, artifacts: 'target/bom.xml', fingerprint: true, onlyIfSuccessful: true
                        }
                    }
                }

                stage('OSS License Checker') {
                    steps {
                        container('licensefinder') {
                            sh '''#!/bin/bash --login
                                /bin/bash --login 
                                rvm use default
                                gem install license_finder
                                license_finder
                            '''.stripIndent()  // Remove indentation to avoid errors
                        }
                    }
                }

                stage('Unit Tests') {
                    steps {
                        container('maven') {
                            sh 'mvn test'
                        }
                    }
                }
            } // End parallel block here
        } 

        stage('SAST') {
            steps {
                container('slscan') {
                    sh 'scan --type java, depscan --build'
                }
            }
            post {
                success {
                    archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/*', fingerprint: true, onlyIfSuccessful: true
                }
            }
        }
        stage('Package') {
            steps {
                container('maven') {
                    sh 'mvn package -DskipTests'
                }
            }
        } 

        stage('OCI Image BnP') {
            steps {
                container('kaniko') {
                    sh '''
                       /kaniko/executor -f `pwd`/Dockerfile -c `pwd` --insecure --skip-tls-verify --cache=true --destination=docker.io/guyenko/dso-demo:latest
                    '''.stripIndent()
                }
            }
        }

        stage('Deploy to Dev') {
            steps {
                // TODO
                sh "echo done"
            }
        }
    }
}
