pipeline {

    agent {
        docker {
            image 'maven:3.9.8-eclipse-temurin-21'
            args  '--entrypoint=""'
        }
    }

    environment {
        APP_NAME    = 'hello-world-2'
        APP_VERSION = "1.0.${env.BUILD_NUMBER}"
        SONAR_URL   = 'http://172.17.0.1:9000'
        MAVEN_OPTS  = '-Dmaven.repo.local=.m2/repository'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
        ansiColor('xterm')
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
                sh 'git log --oneline -5'
            }
        }

        // ── STAGE 2: Build ────────────────────────────────────────────────
        stage('Build') {
            steps {
                echo "Building ${env.APP_NAME} v${env.APP_VERSION}"
                sh 'mvn clean compile -B -Dmaven.test.skip=true'
            }
            post {
                success { echo 'Compile successful — moving to Test stage.' }
                failure { echo 'Compile FAILED — check pom.xml and source errors.' }
            }
        }

        // ── STAGE 3: Test ─────────────────────────────────────────────────
        stage('Test') {
            steps {
                sh 'mvn test -B'
            }
            post {
                always {
                    junit(testResults: 'target/surefire-reports/**/*.xml', allowEmptyResults: true)
                }
                unstable {
                    echo 'WARNING: Tests failed — build marked UNSTABLE.'
                    script {
                        def results = currentBuild.rawBuild.getAction(
                            hudson.tasks.test.AbstractTestResultAction.class)
                        if (results) {
                            def passRate = (results.totalCount - results.failCount) / results.totalCount * 100
                            if (passRate < 80) {
                                error("Test pass rate ${passRate.round(1)}% is below 80% threshold!")
                            }
                        }
                    }
                }
            }
        }

        // ── STAGE 4: Quality Analysis ─────────────────────────────────────
        stage('Quality Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Local') {
                    sh '''
                        mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                          -Dsonar.host.url=http://172.17.0.1:9000 \
                          -Dsonar.projectKey=hello-world-2 \
                          -Dsonar.projectName="TechBuild hello-world-2" \
                          -Dsonar.projectVersion=${APP_VERSION} \
                          -Dsonar.java.binaries=target/classes \
                          -Dsonar.userHome=.m2/.sonar \
                          -B
                    '''
                }
            }
        }

        // ── STAGE 5: Quality Gate ─────────────────────────────────────────
        stage('Quality Gate') {
            agent none
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to Quality Gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }

        // ── STAGE 6: Package & Archive ────────────────────────────────────
        stage('Package & Archive') {
            steps {
                echo "Packaging ${env.APP_NAME} v${env.APP_VERSION}"
                sh "mvn package -DskipTests -B -Drevision=${env.APP_VERSION}"
                archiveArtifacts artifacts: 'target/*.war, target/*.jar', allowEmptyArchive: false, fingerprint: true
                echo "Artifact archived: ${env.APP_NAME}-${env.APP_VERSION}"
            }
            post {
                success { echo 'Package and archive completed successfully.' }
                failure { echo 'Packaging failed — check Maven logs.' }
            }
        }

        // ── STAGE 7: Publish to Nexus ─────────────────────────────────────
        stage('Publish to Nexus') {
            when { branch 'main' }
            steps {
                script {
                    def artifactFile = sh(
                        script: 'ls target/*.war target/*.jar 2>/dev/null | head -n 1',
                        returnStdout: true
                    ).trim()
                    
                    def fileExtension = artifactFile.endsWith('.war') ? 'war' : 'jar'
                    
                    nexusArtifactUploader(
                        nexusVersion:  'nexus3',
                        protocol:      'http',
                        nexusUrl:      '172.17.0.1:8081',
                        groupId:       'io.techbuild',
                        version:       env.APP_VERSION,
                        repository:    'techbuild-releases',
                        credentialsId: 'nexus-creds',
                        artifacts: [[
                            artifactId: env.APP_NAME,
                            classifier: '',
                            file:       artifactFile,
                            type:       fileExtension
                        ]]
                    )
                }
            }
            post {
                success { echo 'Successfully uploaded artifact to Nexus!' }
                failure { echo 'Nexus artifact upload failed.' }
            }
        }
    }

    // ── Post-build actions (Notifications) ───────────────────────────────
    post {
        success {
            echo "PIPELINE SUCCESS — ${env.APP_NAME} v${env.APP_VERSION}"
            slackSend(
                channel: '#ci-notifications',
                color:   'good',
                message: "BUILD PASSED: ${env.APP_NAME} v${env.APP_VERSION} | ${env.BUILD_URL}"
            )
            emailext(
                to:       'devteam@techbuild.io',
                subject:  "BUILD PASSED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body:     "Successful build for ${env.APP_NAME} v${env.APP_VERSION}\nURL: ${env.BUILD_URL}"
            )
        }
        failure {
            echo "PIPELINE FAILED — check logs at ${env.BUILD_URL}"
            slackSend(
                channel: '#ci-notifications',
                color:   'danger',
                message: "BUILD FAILED: ${env.APP_NAME} #${env.BUILD_NUMBER} | ${env.BUILD_URL}"
            )
            emailext(
                to:       'devteam@techbuild.io',
                subject:  "BUILD FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body:     "Build ${env.BUILD_NUMBER} failed.\nConsole: ${env.BUILD_URL}console"
            )
        }
        unstable {
            slackSend(
                channel: '#ci-notifications',
                color:   'warning',
                message: "BUILD UNSTABLE: ${env.APP_NAME} #${env.BUILD_NUMBER} — test failures | ${env.BUILD_URL}"
            )
        }
        always {
            cleanWs()
        }
    }
}