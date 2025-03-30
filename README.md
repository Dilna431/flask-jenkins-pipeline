# Jenkins CI/CD Pipeline for Flask Application Deployment to EC2

This guide explains how to set up a Jenkins pipeline to automate the process of building, testing, and deploying a Flask web application to an AWS EC2 instance.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Install Required Plugins](#install-required-plugins)
- [Configure GitHub Webhook](#configure-github-webhook)
- [Create a Jenkins Pipeline](#create-a-jenkins-pipeline) 
- [Pipeline Script](#pipeline-script) 


## Prerequisites
Before setting up the Jenkins pipeline, ensure you have the following prerequisites:
- A GitHub repository containing the Python web application.
- An AWS EC2 instance with SSH access.
- Jenkins installed and running (using Docker or a dedicated server).
- Python installed on Jenkins and EC2.
- SSH access to your EC2 instance.

## Install Required Plugins
Install the following Jenkins plugins to enable the required features for your pipeline:
1. **Git Plugin**: Allows Jenkins to interact with your GitHub repository.
2. **Pipeline Plugin**: Allows Jenkins to use pipeline scripts for automating workflows.
3. **SSH Pipeline Steps**: Enables Jenkins to SSH into your EC2 instance for deployment.
4. **GitHub Integration Plugin**: Allows Jenkins to integrate with GitHub and trigger builds based on GitHub events (e.g., push events).

## Configure GitHub Webhook
1. Go to **GitHub Repository → Settings → Webhooks**.
2. Add a webhook with the following settings:
   - **Payload URL**: `http://<Jenkins-Server-IP>:8080/github-webhook/`
   - **Content Type**: `application/json`
   - **Trigger**: Select **push events** to trigger the webhook whenever changes are pushed to the repository.

## Create a Jenkins Pipeline
1. Navigate to the Jenkins dashboard and select **New Item** → **Pipeline**.
2. Add the following pipeline script to the Jenkins pipeline configuration:

### Pipeline Script
```groovy
pipeline {
    agent any

    environment {
        APP_DIR = "/home/ubuntu/flask_app"  // Directory on EC2 where the app will be deployed
        SSH_KEY = "/var/jenkins_home/.ssh/id_rsa"
        EC2_USER = "ubuntu"
        EC2_IP = "13.57.48.63"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    credentialsId: '2e869095-a76c-4b00-85e0-cb0fddb3a460',
                    url: 'git@github.com:reshmanavale/FlaskTest.git'
            }
        }

        stage('Build') {
            steps {
                sh '''
                apt update && apt install -y python3-venv python3-pip
                python3 -m venv venv
                . venv/bin/activate
                pip install --upgrade pip
                pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                . venv/bin/activate
                pip install pytest  # Ensure pytest is installed
                pytest || true  # Allow tests to fail without stopping pipeline
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    sh """
                    ssh -o StrictHostKeyChecking=no -i /var/jenkins_home/.ssh/id_rsa ubuntu@13.57.48.63 <<EOF
                    set -e  # Stop script on error
                    
                    echo "🚀 Deploying Flask app..."
                    cd ~/FlaskTest || exit 1

                    # 🔥 STOP any running Flask process before starting a new one
                    echo "🛑 Checking for existing Flask processes..."
                    PID=\$(lsof -t -i:5000) || true  # Prevent failure if no process is running
                    if [ -n "\$PID" ]; then
                        echo "Killing existing Flask process: \$PID"
                        kill -9 \$PID || true  # Ignore failure if the process is already dead
                    else
                        echo "No existing Flask process found."
                    fi

                    # Fetch latest code
                    git reset --hard
                    git pull origin main

                    # Activate virtual environment and install dependencies
                    source venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt

                    # 🚀 Start Flask app in the background
                    echo "Starting Flask app..."
                    nohup venv/bin/python3 app.py > output.log 2>&1 &

                    echo "✅ Deployment completed successfully!"
                    exit 0  # ✅ Ensure Jenkins exits successfully
                    EOF
                    """
                }
            }
        }
    }
}
