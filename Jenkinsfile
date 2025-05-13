pipeline {
  agent any

  environment {
    STAGE_SITE = 'https://stage.bolatitoadegoroye.top'
    AWS_REGION = 'eu-west-2'
  }

  stages {
    stage('Deploy to Staging Environment') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ansible-key', keyFileVariable: 'SSH_KEY'),
          string(credentialsId: 'bastion-id', variable: 'BASTION_INSTANCE_ID'),
          string(credentialsId: 'ansible-ip', variable: 'ANSIBLE_PRIVATE_IP')
        ]) {
          sh '''
            echo "🚀 Starting deployment to staging via SSM Session Manager..."

            # Start SSM SSH tunnel to Bastion
            aws ssm start-session \
              --target "$BASTION_INSTANCE_ID" \
              --document-name "AWS-StartSSHSession" \
              --parameters '{"portNumber":["22"]}' \
              --region "$AWS_REGION" > session.log 2>&1 &

            sleep 5

            # Run Ansible playbook from Ansible server over SSH via tunnel
            ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no -p 22 ubuntu@$ANSIBLE_PRIVATE_IP \
              "ansible-playbook /etc/ansible/playbooks/stage.yml"

            echo "✅ Staging deployment complete."
          '''
        }
      }
    }

    stage('Slack Notification - Staging') {
      steps {
        slackSend channel: 'Cloudhight',
                  message: '✅ New Stage Deployment triggered',
                  teamDomain: '24th-february-sock-shop-kubeadm-project',
                  tokenCredentialId: 'slack'
      }
    }

    stage('DAST Scan') {
      steps {
        sh '''
          echo "🛡️ Running DAST scan with OWASP ZAP..."
          chmod 777 $(pwd)
          docker run -v $(pwd):/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable \
            zap-baseline.py -t $STAGE_SITE -g gen.conf -r testreport.html
        '''
      }
    }

    stage('Prompt for Approval') {
      steps {
        timeout(activity: true, time: 5) {
          input message: '📋 Review before Production Approval', submitter: 'admin'
        }
      }
    }

    stage('Deploy to Production Environment') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ansible-key', keyFileVariable: 'SSH_KEY'),
          string(credentialsId: 'bastion-id', variable: 'BASTION_INSTANCE_ID'),
          string(credentialsId: 'ansible-ip', variable: 'ANSIBLE_PRIVATE_IP')
        ]) {
          sh '''
            echo "🚀 Starting deployment to production via SSM Session Manager..."

            # Start SSM SSH tunnel to Bastion
            aws ssm start-session \
              --target "$BASTION_INSTANCE_ID" \
              --document-name "AWS-StartSSHSession" \
              --parameters '{"portNumber":["22"]}' \
              --region "$AWS_REGION" > session.log 2>&1 &

            sleep 5

            # Run Ansible playbook from Ansible server over SSH via tunnel
            ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no -p 22 ubuntu@$ANSIBLE_PRIVATE_IP \
              "ansible-playbook /etc/ansible/playbooks/playbooks/prod.yml"

            echo "✅ Production deployment complete."
          '''
        }
      }
    }

    stage('Slack Notification - Production') {
      steps {
        slackSend channel: 'Cloudhight',
                  message: '🚀 New Production Deployment completed',
                  teamDomain: '24th-february-sock-shop-kubeadm-project',
                  tokenCredentialId: 'slack'
      }
    }
  }
}
