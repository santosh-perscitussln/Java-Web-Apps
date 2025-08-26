pipeline {
  agent any

  environment {
    GIT_REPO       = 'https://github.com/santosh-perscitussln/Java-Web-Apps.git'
    BRANCH         = 'main'
    GIT_CRED_ID    = 'github-pat'   // Add PAT credentials in Jenkins
    PROD_HOST      = '3.85.162.30'
    PROD_USER      = 'prod-deploy'
    PROD_CRED_ID   = 'ec2-prod-deploy-ssh'
    TOMCAT_WEBAPPS = '/prod/tomcat/apache-tomcat-9.0.99/webapps'
    TOMCAT_BIN = '/prod/tomcat/apache-tomcat-9.0.99/bin'
    APP_NAME       = 'Java-Web-Apps'
    APP_PORT       = '8085'   // Change if Tomcat runs on 8085
  }
  

  stages {
    stage('Checkout') {
      steps {
        git branch: "${BRANCH}", 
            url: "${GIT_REPO}", 
            credentialsId: "${GIT_CRED_ID}"
      }
    }

    stage('Build WAR') {
      steps {
        sh 'mvn -B -DskipTests clean package'
      }
      post {
        success {
          archiveArtifacts artifacts: 'target/*.war', fingerprint: true
        }
      }
    }

    stage('Deploy to EC2') {
      steps {
        sshagent(credentials: [env.PROD_CRED_ID]) {
          sh '''
            WAR_FILE=$(ls target/*.war | head -n1)
            APP_WAR="${APP_NAME}.war"

            mkdir -p ~/.ssh
            chmod 700 ~/.ssh
            ssh-keyscan -H "${PROD_HOST}" >> ~/.ssh/known_hosts

            echo "Stopping Tomcat..."
            #ssh ${PROD_USER}@${PROD_HOST} "sudo systemctl stop tomcat"
            ssh ${PROD_USER}@${PROD_HOST} "${TOMCAT_BIN}/shutdown.sh"
            echo "Removing old deployment..."
            ssh ${PROD_USER}@${PROD_HOST} "rm -rf ${TOMCAT_WEBAPPS}/${APP_NAME} ${TOMCAT_WEBAPPS}/${APP_NAME}.war"

            echo "Copying new WAR..."
            scp "$WAR_FILE" ${PROD_USER}@${PROD_HOST}:${TOMCAT_WEBAPPS}/${APP_WAR}

            echo "Starting Tomcat..."
            #ssh ${PROD_USER}@${PROD_HOST} "sudo systemctl start tomcat"
            ssh ${PROD_USER}@${PROD_HOST} "${TOMCAT_BIN}/startup.sh"
          '''
        }
      }
    }

    stage('Health Check') {
      steps {
        script {
          sh '''
            for i in $(seq 1 20); do
              if curl -fsS "http://${PROD_HOST}:${APP_PORT}/${APP_NAME}/" >/dev/null; then
                echo "✅ App is running!"
                exit 0
              fi
              echo "Waiting for app..."
              sleep 5
            done
            echo "❌ App did not start properly."
            exit 1
          '''
        }
      }
    }
  }
}
