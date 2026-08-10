pipeline {

    agent {
        docker {
            image 'maven:3.9.8-eclipse-temurin-21'
            // Avoid host mount permission issues by removing $HOME/.m2 binding
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
                          -Dsonar.projectName="hello-world-2" \
                          -Dsonar.java.binaries=target/classes \
                          -Dsonar.userHome=.m2/.sonar
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
            steps {
                script {
                    // Detect if the build output is a .jar or .war file
                    def artifactFile = sh(
                        script: 'ls target/*.jar target/*.war 2>/dev/null | head -n 1',
                        returnStdout: true
                    ).trim()
                    
                    def fileExtension = artifactFile.endsWith('.war') ? 'war' : 'jar'
                    
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: '172.17.0.1:8081',
                        groupId: 'io.techbuild',
                        artifactId: 'hello-world-2',
                        version: "1.0.${BUILD_NUMBER}",
                        repository: 'techbuild-releases',
                        credentialsId: 'nexus-creds', // Must match exact ID in Jenkins Credentials
                        artifacts: [
                            [artifactId: 'hello-world-2',
                            classifier: '',
                            file: 'target/hello-world.war',
                            type: 'war']
                        ]
                    )
                }
            }
            post {
                success { echo 'Successfully uploaded artifact to Nexus!' }
                failure { echo 'Nexus artifact upload failed — verify Nexus credentials and repository name.' }
            }
        }

    }

    post {
        always {
            cleanWs()
        }
    }
}