pipeline {

    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from GitHub') {
            steps {

                checkout scmGit(
                    branches: [[name: '*/main']],
                    extensions: [],
                    userRemoteConfigs: [[
                        url: 'https://github.com/satya1031/Book-My-Show.git'
                    ]]
                )

                sh 'ls -la'
            }
        }

        stage('SonarQube Analysis') {
            steps {

                withSonarQubeEnv('sonar-server') {

                    sh """
                    \$SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=BMS \
                    -Dsonar.projectKey=BMS
                    """
                }
            }
        }

        stage('Quality Gate') {

            steps {

                script {

                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Install Dependencies') {

            steps {

                sh '''
                cd bookmyshow-app

                ls -la

                if [ -f package.json ]; then

                    rm -rf node_modules package-lock.json

                    npm install

                else

                    echo "package.json not found!"

                    exit 1
                fi
                '''
            }
        }

        stage('Trivy File Scan') {

            steps {

                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {

            steps {

                script {

                    withDockerRegistry(
                        credentialsId: 'dockerhub-cred'
                    ) {

                        sh '''

                        echo "Building Docker Image..."

                        docker build --no-cache \
                        -t satya1031/bms:latest \
                        -f bookmyshow-app/Dockerfile \
                        bookmyshow-app

                        echo "Pushing Docker Image..."

                        docker push satya1031/bms:latest
                        '''
                    }
                }
            }
        }

        stage('Trivy Image Scan') {

            steps {

                sh 'trivy image satya1031/bms:latest > trivyimage.txt'
            }
        }

        stage('Deploy Container') {

            steps {

                sh '''

                echo "Stopping Old Container..."

                docker stop bms || true

                docker rm bms || true

                echo "Running New Container..."

                docker run -d \
                --restart=always \
                --name bms \
                -p 3000:3000 \
                satya1031/bms:latest

                echo "Checking Running Containers..."

                docker ps -a

                echo "Fetching Logs..."

                sleep 10

                docker logs bms
                '''
            }
        }
    }

    post {

        always {

            emailext(

                attachLog: true,

                subject: "${currentBuild.result}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",

                body: """

                <h2>Jenkins Build Report</h2>

                <p><b>Project:</b> ${env.JOB_NAME}</p>

                <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>

                <p><b>Status:</b> ${currentBuild.result}</p>

                <p><b>Build URL:</b>

                <a href="${env.BUILD_URL}">
                ${env.BUILD_URL}
                </a></p>

                """,

                to: 'emtysoul1031@gmail.com',

                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
    }
}
