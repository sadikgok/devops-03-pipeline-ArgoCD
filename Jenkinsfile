
pipeline {
    agent {
        node {
            label 'My-Jenkins-Agent'
        }
    }
    //agent any
 
    environment {
        APP_NAME = "devops-03-pipeline-aws-gitops"
    }



    stages {


     stage('Cleanup Workspace') {
            steps {
                script {
                    cleanWs()
                }
            }
        }


        stage('SCM GitHub') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/sadikgok/devops-03-pipeline-ArgoCD']])
            }
        }

      

      
stage("Update the Deployment Tags") {
    steps {
        sh """
           echo "Before update:"
           cat deployment.yaml

           sed -i "s|\\(image: *mimaraslan/devops-03-pipeline-aws:\\).*|\\1${IMAGE_TAG}|" deployment.yaml

           echo "After update:"
           cat deployment.yaml
        """
    }
}


stage("Push the changed deployment file to Git") {
  steps {
    withCredentials([usernamePassword(
      credentialsId: 'github',             // Jenkins'teki ID (büyük/küçük harfe dikkat)
      usernameVariable: 'GIT_USER',
      passwordVariable: 'GIT_TOKEN'
    )]) {
      sh '''#!/usr/bin/env bash
                set -euo pipefail

                git config user.name  "sadikgok"
                git config user.email "sadikgok@gmail.com"

# (opsiyonel) detached HEAD ise branch'e geç
                git checkout -B master origin/master || true

                git add deployment.yaml || true

# değişiklik yoksa fail etme
                if git diff --cached --quiet; then
                    echo "No changes to commit."
                exit 0
                fi

                git commit -m "Updated Deployment Manifest"

# Token'ı URL'de kullan; Jenkins maskeler
                git push "https://${GIT_USER}:${GIT_TOKEN}@github.com/sadikgok/devops-03-pipeline-aws-gitops" HEAD:master
'''
    }
  }
}



    }
}

