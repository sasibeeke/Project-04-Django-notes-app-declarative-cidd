# Jenkins CI/CD (Declarative Pipeline) with GitHub Integration (Deploying Django Notes Application)

This documentation outlines the process of setting up a **CI/CD pipeline** using **Jenkins**, **GitHub**, and **Docker** for deploying a **Django Notes Application**. This integration allows for seamless deployments whenever changes are made in the GitHub repository. The guide includes detailed steps and descriptions to ensure that no important information is left out.

---

## Project Overview

In this project, we will:

1. **Create an EC2 instance** on AWS to host Jenkins and Docker.
2. **Set up Jenkins** to automate the CI/CD process.
3. **Use Docker** to containerize the Django application.
4. **Integrate GitHub** for automatic deployments upon code changes.

---

## Steps to Implement the Project

### 1. Create an AWS EC2 Instance

To host the Jenkins server, we first need to create an EC2 instance on AWS.

1. **Login to AWS**:
   - Navigate to the [AWS Console](https://aws.amazon.com) and log in with your credentials.

2. **Launch an EC2 Instance**:
   - Go to the **EC2 Dashboard** and click on **Launch Instance**.
   - Select the **Ubuntu 24.04 LTS** AMI from the list of available Amazon Machine Images (AMIs).
   - Choose the **t2.micro** instance type, which is eligible for the free tier.
   - Click **Next: Configure Instance Details** and proceed through the default settings.
   - In the **Configure Security Group** section, create a new security group allowing **SSH (port 22)** for connecting to the instance and **HTTP (port 80)** for web traffic.

3. **Connect to EC2 Instance**:
   - After launching the instance, you can connect to it using SSH. Open your terminal and run the following command, replacing `<your-key>.pem` with your key file and `<your-ec2-public-ip>` with your instance's public IP address:
   
   ```bash
   ssh -i <your-key>.pem ubuntu@<your-ec2-public-ip>
   ```

---

### 2. Update the EC2 Instance

Before installing any packages, ensure that your instance is up-to-date. This practice helps prevent issues related to outdated packages.

1. **Run the following command**:

   ```bash
   sudo apt update
   ```
   This command updates the package lists for upgrades and new package installations.

---

### 3. Install Java

Since Jenkins is built using Java, you need to install a Java Runtime Environment (JRE) to ensure Jenkins runs correctly.

1. **Install OpenJDK 17**:
   - Run the following command to install the latest version of OpenJDK:
   
   ```bash
   sudo apt install openjdk-17-jre
   ```

2. **Verify Java Installation**:
   - Check whether Java is installed by running:
   
   ```bash
   java -version
   ```

---

### 4. Install Jenkins

Now that Java is installed, you can proceed to install Jenkins, which will be used to create the CI/CD pipeline.

1. **Install Dependencies**:
   - Run the following command to install required dependencies:
   
   ```bash
   sudo apt-get install -y ca-certificates curl gnupg
   ```

2. **Add Jenkins Key**:
   - Execute the following command to add the Jenkins key:
   
   ```bash
   curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
   ```

3. **Add Jenkins Repository**:
   - Add the Jenkins repository to your system:
   
   ```bash
   echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
   ```

4. **Update Package Index**:
   - Run the following command to update the package index:
   
   ```bash
   sudo apt-get update
   ```

5. **Install Jenkins**:
   - Execute the following command to install Jenkins:
   
   ```bash
   sudo apt-get install jenkins
   ```

6. **Enable Jenkins**:
   - Enable Jenkins to start at boot:

   ```bash
   sudo systemctl enable jenkins
   ```

7. **Start Jenkins**:
   - Start the Jenkins service:

   ```bash
   sudo systemctl start jenkins
   ```

8. **Check Jenkins Status**:
   - Verify that Jenkins is running:

   ```bash
   sudo systemctl status jenkins
   ```

9. **Access Jenkins**:
   - Jenkins runs on port 8080. To access Jenkins, go to your EC2 instance's public IP address with port 8080 in your web browser:

   ```
   http://<instance_public_ip>:8080
   ```

10. **Configure Security Group**:
    - In the AWS EC2 dashboard, go to the **Security Group** of your instance, edit the inbound rules, and add a rule for **port 8080**.

---

### 5. Initial Jenkins Setup

1. **Get Admin Password**:
   - To log in to Jenkins for the first time, you will need the initial admin password. Run the following command to retrieve it:

   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```

2. **Login to Jenkins**:
   - Open your web browser and enter the Jenkins URL. Paste the admin password in the required field.

3. **Install Suggested Plugins**:
   - Follow the prompts to install the suggested plugins.

4. **Create Admin User**:
   - Set up an admin user with a password for future logins.

---

### 6. Create a New Jenkins Pipeline Project

1. **Create a New Item**:
   - Click on **New Item** in Jenkins.

2. **Project Title**:
   - Enter a title for your project and select **Pipeline** as the project type, then click **OK**.

3. **Configure GitHub Project**:
   - In the General section, check the **GitHub project** option and provide your GitHub repository URL.

4. **Create a Jenkinsfile**:
   - In your GitHub repository, create a file named `Jenkinsfile` and add the following pipeline script:

   ```groovy
   pipeline {
       agent any
       stages {
           stage("Clone The code...") {
               steps {
                   echo "Cloning the code"
                   git url: "<Your_github_project_repo_url>", branch: "main"
               }
           }
           stage("Build and Test...") {
               steps {
                   echo "Building the Docker image(Container)"
                   sh "docker build . -t cicd-note-app:latest"
               }
           }
           stage("Push build to Docker Hub") {
               steps {
                   echo "Pushing build to DockerHub..."
                   withCredentials([
                       usernamePassword(
                           credentialsId: "dockerHub",
                           passwordVariable: "dockerHubPass",
                           usernameVariable: "dockerHubUser"
                       )    
                   ]) {
                       sh "docker tag cicd-note-app ${env.dockerHubUser}/cicd-note-app:latest"
                       sh "docker login -u ${env.dockerHubUser} -p ${env.dockerHubPass}"
                       sh "docker push ${env.dockerHubUser}/cicd-note-app:latest"
                   }
               }
           }
           stage("Deploy the Container") {
               steps {
                   echo "Deploying docker Container..."
                   sh "docker compose down && docker compose up -d"
               }
           }
       }
   }
   ```

5. **Save the Jenkins Project**:
   - Click on **Save** after configuring your pipeline project.

---

### 7. Configure Docker Hub

1. **Create Docker Hub Account**:
   - If you don't already have a Docker Hub account, create one at [Docker Hub](https://hub.docker.com/).

2. **Add Credentials in Jenkins**:
   - In Jenkins, go to **Manage Jenkins** -> **Manage Credentials** -> **System** -> **Global credentials (unrestricted)** -> **Add Credentials**.
   - Select **Username with Password** and enter your Docker Hub credentials. Provide an ID name (e.g., `dockerHub`) to identify these credentials.

---

### 8. Install Docker and Docker Compose

1. **Install Docker**:
   - Run the following command to install Docker:

   ```bash
   sudo apt install docker.io
   ```

2. **Add Jenkins User to Docker Group**:
   - This allows Jenkins to run Docker commands without needing sudo:

   ```bash
   sudo usermod -aG docker jenkins
   ```

3. **Reboot the Server**:
   - Reboot the server to apply changes:

   ```bash
   sudo reboot
   ```

4. **Install Docker Compose**:
   - Follow these steps to install Docker Compose:

   ```bash
   sudo apt-get update
   sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt-get update
   sudo apt-get install docker-compose
   ```

5. **Verify Docker and Docker Compose Installation**:
   - Check the versions to ensure they are installed correctly:
   
   ```bash
   docker --version
   docker-compose --version
   ```

---

### 9. Deploy the Application

1. **Trigger a Build**:
   - Go back to your Jenkins pipeline project and click **Build Now** to start the pipeline.

2. **Monitor the Console Output**:
   - Click on the build number to view the console output and monitor the deployment process.

3. **Access the Application**:
   - Once the build is complete, access your Django Notes Application using the public IP of your EC2 instance.

---

### Conclusion

You have successfully set up a CI/CD pipeline using Jenkins, GitHub, and Docker for your Django Notes Application. This setup enables automatic deployments whenever changes are pushed to the GitHub repository, streamlining the development and deployment process.

For further enhancements, consider exploring:

- **Adding automated tests in your pipeline**: Implement unit and integration tests to ensure code quality before deployment.
- **Implementing notifications for build status**: Configure email or Slack notifications to keep the team informed about build successes or failures.
- **Configuring additional stages based on your project's requirements**: Explore other Jenkins capabilities, such as testing and security checks, to further improve your pipeline.
- **Setting up monitoring and logging for your application**: Use tools like Prometheus, Grafana, or ELK Stack to monitor application performance and log activities.

With this CI/CD pipeline, you can ensure that your Django Notes Application is continuously integrated and delivered with high reliability and efficiency.


# Output Images of Project ( Which I have done while practicing CI/CD for this Project which ensure that it works properly )

1 ) Commiting the changes on Github Repo :

![Screenshot 2024-09-24 121349](https://github.com/user-attachments/assets/e6157c86-e806-43d0-8446-4b0cb58d71e4)

2 ) It successully Trigger the pipeline on Github push :

![Screenshot 2024-09-24 120930](https://github.com/user-attachments/assets/a300c1d9-865e-4977-b3f0-cc90fe56405c)

3 ) Running the Pipeline Script :

![Screenshot 2024-09-24 120937](https://github.com/user-attachments/assets/6cdff4b5-2c10-49f2-aedb-9ae7656cc365)

4 ) Successfully Run the Pipeline script :

![Screenshot 2024-09-24 120954](https://github.com/user-attachments/assets/64b4eeaf-9270-4fff-a762-0daa4d3044ca)

5 ) Viewing Stages of that Jenkins Job ( Which works properly ) :

![Screenshot 2024-09-24 121014](https://github.com/user-attachments/assets/dcb8d5c9-0f16-4071-aff4-371370ca602a)

6 ) Docker hub Repositary successfully created using pipeline script :

![Screenshot 2024-09-24 121920](https://github.com/user-attachments/assets/fce9bd0d-52bb-42d6-9874-250d131c64a3)

7 ) Output of The Project ( All features of it working properly ) :

![Screenshot 2024-09-24 121659](https://github.com/user-attachments/assets/347fa289-b7b6-4753-843e-02b0029e99cc)  

