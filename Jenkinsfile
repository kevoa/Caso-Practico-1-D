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
                    # Instalar las herramientas necesarias.
                    # flake8-html es obsoleto, así que lo quitamos.
                    pip install flake8 bandit --break-system-packages
        
                    # Corregimos el PATH para que los comandos sean encontrados.
                    export PATH="\$PATH:/home/jenkins/.local/bin"
                    
                    # flake8 genera su informe.
                    # Usamos un formato simple y redirigimos la salida a un archivo.
                    flake8 ./src --output-file=flake8-report.txt || true
                    
                    # bandit funciona correctamente, lo dejamos como está.
                    bandit -r ./src -f html -o bandit-report.html || true
                """
        
                echo "Publicando informes..."
                // Para flake8, publicamos el archivo de texto.
                // La práctica pide publicar un informe, y este archivo cumple la función.
                // El plugin de htmlpublisher puede publicar cualquier archivo.
                publishHTML(target: [ reportDir: '.', reportFiles: 'flake8-report.txt', reportName: 'Informe Flake8' ])
                publishHTML(target: [ reportDir: '.', reportFiles: 'bandit-report.html', reportName: 'Informe Bandit' ])
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo "Empaquetando y desplegando en Staging con SAM dentro de un contenedor..."
                sh 'sam build'
                
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
                sh 'pip install -r test/integration/requirements.txt --break-system-packages'

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