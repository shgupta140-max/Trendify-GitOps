pipeline {
    agent any

    environment {
        CLUSTER_NAME = 'trendify-cluster'
        AWS_REGION = 'ap-south-1'
        SLACK_NOTIFICATION_CHANNEL = '#trendify-notifications'
        SLACK_APPROVAL_CHANNEL = '#trendify-approvals'
    }
    stages {
        stage('Get the .kube/config file') {
            steps {
                echo 'Getting the .kube/config file...'
                sh "aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}"
            }
        }
        stage('Checking kubectl version') {
            steps {
                echo 'Checking kubectl version...'
                sh 'kubectl version'
            }
        }
        stage('Deploying Manifests via kustomize') {
            options {
                timeout(time: 1, unit: 'HOURS')
            }
           steps {
                slackSend(
                    message: "Approval to deploy manifests on ${CLUSTER_NAME} in region ${AWS_REGION} for job\nJob:${env.JOB_NAME} - Build #${env.BUILD_NUMBER}\nPlease approve the deployment here: ${env.BUILD_URL}console",
                    channel: "${SLACK_APPROVAL_CHANNEL}",
                    color: '#FFFF00'
                )

                input (
                    message: "Do you want to deploy manifests on ${CLUSTER_NAME} in region ${AWS_REGION}?",
                    ok: 'Deploy',
                    cancel: 'Abort'
                )
                echo 'Deploying...'
                sh 'kubectl apply -k ./kubedefs'
            }
        }

        stage('Checking Deployment Status') {
            steps {
                echo 'Checking deployment status...'
                sh 'kubectl rollout status deployment trendstore-deployment'
                echo 'Checking Pod status...'
                sh 'kubectl get pods -n monitoring -o wide'
            }
        }
    }

    post {
        success {
            slackSend(
                message: "Deployment successful on ${CLUSTER_NAME} in region ${AWS_REGION} for job\nJob:${env.JOB_NAME} - Build #${env.BUILD_NUMBER}\nCheck the build details here: ${env.BUILD_URL}",
                channel: "${SLACK_NOTIFICATION_CHANNEL}",
                color: '#00FF00'
            )
        }

        failure {
            slackSend(
                message: "Deployment failed on ${CLUSTER_NAME} in region ${AWS_REGION} for job\nJob:${env.JOB_NAME} - Build #${env.BUILD_NUMBER}\nCheck the build details here: ${env.BUILD_URL}",
                channel: "${SLACK_NOTIFICATION_CHANNEL}",
                color: '#FF0000'
            )
        }

        aborted {
            script {
                def approver = env.ABORTED_BY ?: 'Unknown user'

                slackSend(
                    message: "Deployment aborted by ${approver} on ${CLUSTER_NAME} in region ${AWS_REGION} for job\nJob:${env.JOB_NAME} - Build #${env.BUILD_NUMBER}\nCheck the build details here: ${env.BUILD_URL}",
                    channel: "${SLACK_NOTIFICATION_CHANNEL}",
                    color: '#FFA500'
                )
            }
        }
    }
}