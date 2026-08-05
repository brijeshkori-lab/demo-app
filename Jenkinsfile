pipeline {
    agent any

    environment {
         JFROG_USER  = credentials('jfrog-user')
        JFROG_TOKEN = credentials('jfrog-token')
        SONAR_TOKEN = credentials('sonar-token')
        PROJECT     = 'demo-app'
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Configure JFrog CLI') {
            steps {
                sh """
                    jf config add ${PROJECT}-server \
                        --url=https://trialjpunep.jfrog.io \
                        --user=\$JFROG_USER \
                        --access-token=\$JFROG_TOKEN \
                        --interactive=false --overwrite=true

                    jf mvnc \
                        --repo-resolve-releases=${PROJECT}-maven-virtual \
                        --repo-resolve-snapshots=${PROJECT}-maven-virtual \
                        --repo-deploy-releases=${PROJECT}-libs-release-local \
                        --repo-deploy-snapshots=${PROJECT}-libs-snapshot-local \
                        --server-id-resolve=${PROJECT}-server \
                        --server-id-deploy=${PROJECT}-server
                """
            }
        }

        stage('Build') {
            steps { sh 'jf mvn -U clean compile' }
        }

        stage('Unit Tests') {
            steps { sh 'jf mvn test' }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQubeLocal') {
                    sh """
                        jf mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.10.0.2594:sonar \
                            -Dsonar.projectKey=${PROJECT} \
                            -Dsonar.token=\$SONAR_TOKEN
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package') {
            steps {
                sh 'jf mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Deploy & Publish Build Info') {
            steps {
                sh '''
                    jf mvn deploy -DskipTests
                    jf rt build-publish
                '''
            }
        }
    }

    post {
        success { echo "✅ demo-app pipeline succeeded — Build #${BUILD_NUMBER}" }
        failure { echo "❌ demo-app pipeline failed" }
    }
}