pipeline {
    agent any

    environment {
        IMAGE_NAME = "nour0205/my_app"
        BUILD_TAG = "${BUILD_NUMBER}"
    }

    stages {

        

        stage('Cleanup') {
            steps {
                bat '''
                    echo [CLEANUP] Stopping and removing previous Docker Compose containers...
                    docker compose down -v || exit 0

                    echo [CLEANUP] Forcibly removing specific containers if still running...
                    for /f %%i in ('docker ps -a -q --filter "name=myapppipeline-web-1"') do docker rm -f %%i
                    for /f %%i in ('docker ps -a -q --filter "name=myapppipeline-postgres-1"') do docker rm -f %%i
                    for /f %%i in ('docker ps -a -q --filter "name=myapppipeline-mongo-1"') do docker rm -f %%i

                    echo [CLEANUP] Killing any process locking port 5001 (web)...
                    for /f "tokens=5" %%i in ('netstat -aon ^| findstr :5001 ^| findstr LISTENING') do taskkill /PID %%i /F

                    echo [CLEANUP] Killing any process locking port 5432 (PostgreSQL)...
                    for /f "tokens=5" %%i in ('netstat -aon ^| findstr :5432 ^| findstr LISTENING') do taskkill /PID %%i /F

                    echo [CLEANUP] Killing any process locking port 27017 (MongoDB)...
                    for /f "tokens=5" %%i in ('netstat -aon ^| findstr :27017 ^| findstr LISTENING') do taskkill /PID %%i /F

                    echo [CLEANUP] Cleanup complete.
                '''
            }
        }

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/nour0205/devops_my_app.git'
            }
        }

        stage('Install Dependencies & Generate Prisma Client') {
            steps {
                bat 'npm install'
                bat 'dir node_modules\\.bin'
                bat 'npx prisma generate'
            }
        }

        stage('Run Backend Tests') {
            steps {
                bat 'npm run test'
            }
            post {
                always {
                    junit 'reports/junit.xml'
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    bat 'npm install'
                    bat 'npm run build'
                }
                bat 'rmdir /S /Q public || exit 0'
                bat 'move frontend\\dist public'
            }
        }

       stage('Build Docker Image') {
    steps {
        bat """
            docker build -t ${IMAGE_NAME}:${BUILD_TAG} .
            docker tag ${IMAGE_NAME}:${BUILD_TAG} ${IMAGE_NAME}:latest
        """
    }
}

stage('Push Image to Docker Hub') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
            bat """
                docker login -u %DOCKER_USERNAME% -p %DOCKER_PASSWORD%
                docker push ${IMAGE_NAME}:${BUILD_TAG}
                docker push ${IMAGE_NAME}:latest
            """
        }
    }
}

        stage('Deploy with Docker Compose') {
            steps {
                bat 'docker-compose down || exit 0'
                bat 'docker-compose up -d --build'
            }
        }

        stage('Run Prisma Migrate') {
    steps {
        bat '''
@echo off
setlocal enabledelayedexpansion
echo 🔄 Waiting for PostgreSQL container to become healthy...

set RETRIES=10
set WAIT=3
set ATTEMPT=1

:wait_loop
for /f %%H in ('docker inspect --format="{{.State.Health.Status}}" myapppipeline-postgres-1') do (
    if "%%H"=="healthy" (
        echo ✅ PostgreSQL is healthy!
        goto :migrate
    )
)

echo ⏳ Attempt !ATTEMPT! of %RETRIES%: Not healthy yet...
set /a ATTEMPT+=1
if !ATTEMPT! GTR %RETRIES% (
    echo ❌ PostgreSQL did not become healthy after %RETRIES% attempts.
    exit /b 1
)

timeout /T %WAIT% >nul
goto :wait_loop

:migrate
echo 🚀 Running Prisma Migrate Deploy...
docker exec myapppipeline-web-1 npx prisma migrate deploy
'''
    }
}


        stage('Code Quality - SonarQube') {
            steps {
                withCredentials([string(credentialsId: 'jenkins-sonar', variable: 'SONAR_TOKEN')]) {
                    bat 'npm test'
                    bat '"C:\\Program Files\\sonar-scanner-7.1.0.4889-windows-x64\\bin\\sonar-scanner.bat" -Dsonar.token=%SONAR_TOKEN%'
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful!"
        }
        failure {
            echo "❌ Deployment failed!"
        }
    }
}
