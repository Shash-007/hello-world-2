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

        // ── STAGE 8: Deploy Application ───────────────────────────────────
        stage('Deploy Application') {
            // Override top-level container: run on host OS directly
            agent { node { label '' } } 
            steps {
                // Copy archived artifact from Stage 6 into host environment
                unarchive mapping: ['target/*.war': '.', 'target/*.jar': '.']
                
                script {
                    echo "Deploying ${env.APP_NAME} on Host Port 8082..."
                    
                    sh '''
                        # 1. Locate the unarchived artifact
                        WAR_FILE=$(ls target/*.war target/*.jar 2>/dev/null | head -n 1)
                        if [ -z "$WAR_FILE" ]; then
                            echo "ERROR: No artifact found after unarchive step."
                            exit 1
                        fi
                        
                        echo "Copying $WAR_FILE to /var/tmp/app.war"
                        cp "$WAR_FILE" /var/tmp/app.war

                        # 2. Kill existing process running on port 8082
                        PID=$(lsof -ti:8082 || true)
                        if [ -n "$PID" ]; then
                            echo "Stopping existing process on port 8082 (PID: $PID)..."
                            kill -9 $PID || true
                        fi

                        # 3. Start background app process on Host OS
                        JENKINS_NODE_COOKIE=dontKillMe nohup java -jar /var/tmp/app.war --server.port=8082 > /var/tmp/app.log 2>&1 &
                        
                        sleep 5
                        echo "Deployment command executed."
                    '''
                }
            }
            post {
                success { echo "Deployment complete! App running at http://15.168.173.29:8082" }
                failure { echo "Deployment failed." }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}