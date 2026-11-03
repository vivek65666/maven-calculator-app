// CI/CD pipeline for the Java Maven Calculator Web App (Windows Compatible).
//
// Stage order: Checkout -> Validate & Lint -> Build -> Unit Test -> SonarQube Analysis -> Quality Gate
// -> Package -> Integration Test -> Publish to Artifactory
// -> Deploy to QA -> Performance Test (on QA) -> Approval -> Deploy to Production

pipeline {
    agent any

    parameters {
        booleanParam(name: 'RUN_SONAR', defaultValue: true, description: 'Run SonarQube analysis and Quality Gate')
    }

    environment {
        ARTIFACTORY_URL  = 'http://localhost:8082/artifactory'
        QA_TOMCAT        = 'http://localhost:8080'
        PROD_TOMCAT      = 'http://localhost:8080'
        QA_CONTEXT       = 'calculator-qa'
        PROD_CONTEXT     = 'calculator'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate & Lint') {
            steps {
                bat 'mvn -B validate checkstyle:check'
            }
        }

        stage('Build') {
            steps {
                bat 'mvn -B clean compile'
            }
        }

        stage('Unit Test') {
            steps {
                bat 'mvn -B test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            when { expression { params.RUN_SONAR } }
            steps {
                withSonarQubeEnv('sonarqube') {
                    bat 'mvn -B sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            when { expression { params.RUN_SONAR } }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package') {
            steps {
                bat 'mvn -B package -DskipTests'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/calculator.war', fingerprint: true
                }
            }
        }

        stage('Integration Test') {
            steps {
                bat 'mvn -B verify -DskipUTs'
            }
            post {
                always {
                    junit 'target/failsafe-reports/*.xml'
                }
            }
        }

        stage('Publish to Artifactory') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'artifactory-creds',
                        usernameVariable: 'ARTIFACTORY_USERNAME', passwordVariable: 'ARTIFACTORY_PASSWORD')]) {
                    bat 'mvn -B -s settings-artifactory.xml deploy -DskipTests -Dartifactory.url=%ARTIFACTORY_URL%'
                }
            }
        }

        stage('Deploy to QA') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'tomcat-creds',
                        usernameVariable: 'TOMCAT_USERNAME', passwordVariable: 'TOMCAT_PASSWORD')]) {
                    bat 'mvn -B cargo:redeploy -Dtomcat.url=%QA_TOMCAT%/manager/text -Dtomcat.path=%QA_CONTEXT%'
                }
                bat 'curl -sf --retry 10 --retry-delay 3 --retry-all-errors %QA_TOMCAT%/%QA_CONTEXT%/api/calculator/ping'
            }
        }

        stage('Performance Test') {
            steps {
                bat '''
                    mvn -B verify -Pperformance -DskipTests ^
                        -Dperf.host=localhost -Dperf.port=8080 -Dperf.context=%QA_CONTEXT%
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'target/jmeter/reports/**', allowEmptyArchive: true
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'QA and performance tests passed. Deploy to Production?', ok: 'Deploy'
                }
            }
        }

        stage('Deploy to Production') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'tomcat-creds',
                        usernameVariable: 'TOMCAT_USERNAME', passwordVariable: 'TOMCAT_PASSWORD')]) {
                    bat 'mvn -B cargo:redeploy -Dtomcat.url=%PROD_TOMCAT%/manager/text -Dtomcat.path=%PROD_CONTEXT%'
                }
                bat 'curl -sf --retry 10 --retry-delay 3 --retry-all-errors %PROD_TOMCAT%/%PROD_CONTEXT%/api/calculator/ping'
            }
        }
    }

    post {
        success {
            echo "Deployed: ${env.PROD_TOMCAT}/${env.PROD_CONTEXT}/api/calculator/ping"
        }
        failure {
            echo 'Pipeline failed - check the red stage above.'
        }
    }
}