pipeline {
    // Le decimos a Jenkins que ejecute todo en nuestro agente con la etiqueta 'general'
    agent { label 'general' }

    environment {
        AWS_REGION         = 'us-east-1'
        // ¡IMPORTANTE! Reemplaza esto con el nombre del bucket S3 que creaste
        S3_BUCKET_NAME     = 'aws-sam-cli-managed-default-samclisourcebucket-bzstlrjji7cw' 
        STAGING_STACK_NAME = 'todo-api-staging'
    }

    stages {
        stage('Get Code') {
            steps {
                echo "Descargando código de la rama ${env.BRANCH_NAME}..."
                sh 'mkdir -p ~/.ssh'
                sh 'ssh-keyscan github.com >> ~/.ssh/known_hosts'
                sh 'git config --global core.sshCommand "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"'

                
                checkout scm
            }
        }

        stage('Static Test') {
            steps {
                echo "Ejecutando análisis estático..."
                sh """
                    # Combinamos todos los comandos en un solo bloque sh
                    # para que la variable PATH se aplique correctamente.
                    
                    pip install flake8 bandit --break-system-packages
                    export PATH="\$PATH:/home/jenkins/.local/bin"
                    
                    # flake8 y bandit ahora se ejecutan con el PATH correcto
                    flake8 ./src --format=html --htmldir=flake8-report || true
                    bandit -r ./src -f html -o bandit-report.html || true
                """
        
                echo "Publicando informes..."
                publishHTML(target: [ reportDir: 'flake8-report', reportFiles: 'index.html', reportName: 'Informe Flake8' ])
                publishHTML(target: [ reportDir: '.', reportFiles: 'bandit-report.html', reportName: 'Informe Bandit' ])
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo "Empaquetando y desplegando en Staging con SAM dentro de un contenedor..."
                sh 'sam build --use-container'
                sh """
                    sam deploy \\
                        --stack-name ${STAGING_STACK_NAME} \\
                        --s3-bucket ${S3_BUCKET_NAME} \\
                        --capabilities CAPABILITY_IAM \\
                        --region ${AWS_REGION} \\
                        --no-fail-on-empty-changeset
                """
            }
        }

        stage('Rest Test (Pytest)') {
            steps {
                echo "Instalando dependencias para las pruebas..."
                sh 'pip install -r test/integration/requirements.txt'

                echo "Obteniendo la URL del API Gateway..."
                script {
                    def apiUrl = sh(
                        script: "aws cloudformation describe-stacks --stack-name ${STAGING_STACK_NAME} --query 'Stacks[0].Outputs[?OutputKey==`TodoApi`].OutputValue' --output text --region ${AWS_REGION}",
                        returnStdout: true
                    ).trim()
                    env.API_URL = apiUrl // Guarda la URL en una variable de entorno para usarla más adelante
                }
                
                echo "Ejecutando pruebas de integración contra: ${env.API_URL}"
                sh "API_URL=${env.API_URL} pytest test/integration/todoApiTest.py"
            }
        }

        stage('Promote') {
            steps {
                echo "Promocionando el código a la rama 'master'..."
                // Usamos la credencial SSH que ya configuraste para tu repositorio
                withCredentials([sshUserPrivateKey(credentialsId: 'github-ssh-key', keyFileVariable: 'GIT_SSH_KEY')]) {
                    sh """
                        # Configura Git para usar la clave SSH
                        mkdir -p ~/.ssh
                        cp \$GIT_SSH_KEY ~/.ssh/id_ed25519
                        chmod 600 ~/.ssh/id_ed25519
                        ssh-keyscan github.com >> ~/.ssh/known_hosts

                        # Configura el autor del commit
                        git config --global user.email "jenkins@kevin.com"
                        git config --global user.name "Jenkins CI"
                        
                        # Realiza el merge y el push
                        git checkout master
                        git merge develop
                        git push origin master
                    """
                }
            }
        }
    }
}