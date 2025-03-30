# CI/CD Pipeline using Jenkins and GitHub Actions

This guide provides step-by-step instructions to set up a CI/CD pipeline using Jenkins and GitHub Actions for a Python web application. The pipeline includes build, test, and deployment to an AWS EC2 instance.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Setup Jenkins](#setup-jenkins)
3. [Configure GitHub Actions](#configure-github-actions)
4. [CI/CD Workflow](#cicd-workflow)
5. [Deployment to AWS EC2](#deployment-to-aws-ec2)
6. [Screenshots and Attachments](#screenshots-and-attachments)
7. [Troubleshooting](#troubleshooting)
8. [Conclusion](#conclusion)

---

## Prerequisites
- A GitHub repository containing the Python web application.
- An AWS EC2 instance with SSH access.
- Jenkins installed and running (using Docker or a dedicated server).
- GitHub Actions enabled on your repository.
- Python installed on Jenkins and EC2.

---

## Setup Jenkins

### 1. Install Jenkins
You can install Jenkins on a virtual machine, use a cloud-based Jenkins service, or run Jenkins via Docker.
If running Jenkins via Docker:
#### Install Jenkins on Docker  
mkdir jenkins_home && cd jenkins_home  
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 -v $(pwd):/var/jenkins_home jenkins/jenkins:lts  

### 2. Install Required Plugins
- **Git Plugin**
- **Pipeline Plugin**
- **SSH Pipeline Steps** (for deployment to EC2)
- **GitHub Integration Plugin**

### 3. Configure GitHub Webhook
- Go to **GitHub Repository → Settings → Webhooks**
- Add a webhook with the URL: `http://<Jenkins-Server-IP>:8080/github-webhook/`
- Set content type to `application/json`
- Trigger on `push` events

### 4. Create a Jenkins Pipeline
1. Navigate to **Jenkins Dashboard → New Item → Pipeline**
2. Add the following pipeline script:

```groovy
pipeline {
    agent any
    environment {
        APP_DIR = "flask_app"
        SSH_KEY = "/var/jenkins_home/.ssh/id_rsa"
        EC2_USER = "ubuntu"
        EC2_IP = "13.57.8.246"
    }
    stages {
        stage('Checkout Code') {
            steps {
                dir("${WORKSPACE}/${APP_DIR}") {
                    git branch: 'main',
                        url: 'https://github.com/aakashrawat1910/CICDFlaskTest.git'
                }
            }
        }
        stage('Build') {
            steps {
                dir("${WORKSPACE}/${APP_DIR}") {
                    sh '''
                    apt update && apt install -y python3-venv python3-pip
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    '''
                }
            }
        }
        stage('Test') {
            steps {
                dir("${WORKSPACE}/${APP_DIR}") {
                    sh '''
                    . venv/bin/activate
                    pip install pytest
                    pytest || true
                    '''
                }
            }
        }
        stage('Deploy to EC2') {
            steps {
                script {
                    sh """
                    ssh -o StrictHostKeyChecking=no -i ${SSH_KEY} ${EC2_USER}@${EC2_IP} <<EOF
                    set -e  # Stop on error

                    # Create application directory if it doesn't exist
                    mkdir -p ${APP_DIR}

                    # Navigate to application directory
                    cd ${APP_DIR} || exit 1

                    # Clone the repository if it doesn't exist
                    if [ ! -d ".git" ]; then
                        git clone https://github.com/aakashrawat1910/CICDFlaskTest.git .
                    fi

                    # Pull the latest code from the repository
                    git pull origin main

                    # Set up virtual environment and install dependencies
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt

                    # Run the Flask application
                    nohup python3 -m app &

                    exit
                    EOF
                    """
                }
            }
        }
    }
}
```

---

## Configure GitHub Actions

### 1. Create a GitHub Actions Workflow
1. Inside your repository, navigate to `.github/workflows/`
2. Create a new file `ci-cd.yml` and add the following:

```yaml
name: Flask CI/CD Pipeline

on:
  push:
    branches:
      - main
      - staging
  pull_request:
    branches:
      - main
      - staging

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: 3.9

      - name: Install Dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run Tests
        run: python -m unittest discover

  deploy-staging:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Deploy to Staging Server
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          STAGING_HOST: ${{ secrets.STAGING_HOST }}
          STAGING_USER: ${{ secrets.STAGING_USER }}
      
        run: |
          echo "$SSH_PRIVATE_KEY" > deploy_key.pem
          chmod 600 deploy_key.pem

          ssh -o StrictHostKeyChecking=no -i deploy_key.pem $STAGING_USER@$STAGING_HOST << 'EOF'
            set -e  # Exit on error
            
            # Ensure project directory exists
            mkdir -p /home/ubuntu/FlaskTest
            cd /home/ubuntu/FlaskTest

            # Clone repo if it doesn't exist
            if [ ! -d ".git" ]; then
              git clone git@github.com:your-user/your-repo.git .
            fi

            # Ensure correct branch
            git fetch origin
            if git rev-parse --verify staging; then
              git checkout staging
            else
              git checkout -b staging
            fi
            git pull origin staging

            # Ensure virtual environment exists
            if [ ! -d "venv" ]; then
              python3 -m venv venv
            fi
            source venv/bin/activate

            # Check if requirements.txt exists before installing dependencies
            if [ -f "requirements.txt" ]; then
              pip install -r requirements.txt
            else
              echo "ERROR: requirements.txt not found!"
              exit 1
            fi

            # Restart Flask application
            if systemctl list-units --full -all | grep -Fq "flaskapp.service"; then
              sudo systemctl restart flaskapp
            else
              echo "Warning: flaskapp.service not found. Ensure it's set up."
            fi
          EOF
 ```

### 2. Set Up Secrets in GitHub
- **EC2_HOST**: Public IP of your EC2 instance
- **EC2_SSH_KEY**: Private SSH key for authentication

---

## CI/CD Workflow
1. **Code Push**: Developers push code to the `main` branch.
2. **GitHub Actions**: Triggers build, test, and deploy steps.
3. **Jenkins**: Pulls the latest code, tests, and deploys it to EC2.
4. **Deployment**: Application is updated on the EC2 instance.

---

## Deployment to AWS EC2
1. Connect to your EC2 instance:
   ```sh
   ssh -i your-key.pem ubuntu@your-ec2-ip
   ```
2. Ensure the necessary packages are installed:
   ```sh
   sudo apt update && sudo apt install python3-pip git -y
   ```
3. Clone the repository:
   ```sh
   git clone https://github.com/your-repo.git /var/www/app
   ```
4. Setup a systemd service:
   ```sh
   sudo nano /etc/systemd/system/myapp.service
   ```
   Add the following:
   ```ini
   [Unit]
   Description=My Python App
   After=network.target

   [Service]
   User=ubuntu
   WorkingDirectory=/var/www/app
   ExecStart=/usr/bin/python3 /var/www/app/app.py
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```
5. Start and enable the service:
   ```sh
   sudo systemctl daemon-reload
   sudo systemctl enable myapp.service
   sudo systemctl start myapp.service
   ```

---
## Troubleshooting
### Common Issues & Fixes
- **Permission Denied (SSH to EC2)**:
  ```sh
  chmod 400 your-key.pem
  ```
- **GitHub Actions Fails at SSH Connection**:
  - Ensure the correct SSH private key is added as a GitHub secret.
- **Jenkins Not Triggering Builds**:
  - Verify the webhook setup in GitHub settings.

---

## Conclusion
This guide provides a fully automated CI/CD pipeline integrating Jenkins and GitHub Actions for deploying a Python web application to AWS EC2. 

