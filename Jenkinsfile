pipeline {

    agent {
        docker {
            image 'maven:3.9.8-eclipse-temurin-21'
            args  '-v $HOME/.m2:/tmp/.m2 -e MAVEN_CONFIG=/tmp/.m2'
        }
    }

    environment {
        APP_NAME    = 'hello-world-2'
        APP_VERSION = "1.0.${env.BUILD_NUMBER}"
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
    }

    triggers {
        githubPush()
    }

    stages {

        // ── STAGE 1: Checkout ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch: ${env.GIT_BRANCH} | Commit: ${env.GIT_COMMIT[0..7]}"
            }
        }

        // ── STAGE 2: Build ────────────────────────────────────────────────
        stage('Build') {
            steps {
                echo "Building ${env.APP_NAME} v${env.APP_VERSION}"
                sh 'mvn clean compile -B -Dmaven.test.skip=true -Duser.home=/tmp'
            }
            post {
                success { echo 'Compile successful — moving to Test stage.' }
                failure { echo 'Compile FAILED — check pom.xml and source errors.' }
            }
        }

        // ── STAGE 3: Test ─────────────────────────────────────────────────
        stage('Test') {
            steps {
                sh 'mvn test -B -Duser.home=/tmp'
            }
            post {
                always {
                    junit(testResults: 'target/surefire-reports/**/*.xml', allowEmptyResults: true)
                }
            }
        }

        // ── STAGE 4: Quality Analysis ─────────────────────────────────────
        stage('Quality Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Local') {
                    sh '''
                        mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                          -Duser.home=/tmp \
                          -Dsonar.projectKey=hello-world-2 \
                          -Dsonar.projectName="hello-world-2" \
                          -Dsonar.java.binaries=target/classes
                    '''
                }
            }
            post {
                success { echo 'SonarQube analysis submitted successfully.' }
                failure { echo 'SonarQube analysis FAILED — check connection and credentials.' }
            }
        }

    }

    post {
        always {
            cleanWs()
        }
    }
}