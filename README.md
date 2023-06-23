# cicddemo
Prerequisites:
- Basic understanding of GitHub Actions, Docker, and AWS EC2
- A GitHub account
- A DockerHub account
- An AWS account
- An EC2 instance running Ubuntu with Docker installed

Hands on with Continuous Deployment

Image: a simple Continuous Deployment Process
Directory Structure:
```
.
├── .github
│ └── workflows
│ └── ec2deploy.yaml
├── app.js
├── Dockerfile
├── docker-compose.yml
└── package.json
```

Find all the codes here.

Step 1: Create a simple Express.js application
Create a new directory for your project and navigate to it in your terminal. Create the following files with the content provided:

app.js

const express = require('express');
const app = express();
app.get('/', (req, res) => {
 res.send('Hello, World!');
});
app.listen(3000, () => {
 console.log('Server running on port 3000');
});
package.json

{
 "name": "cd-demo",
 "version": "1.0.0",
 "description": "Continuous Deployment with GitHub Actions, DockerHub, and AWS EC2 tutorial for beginners",
 "scripts": {
 "start": "node app.js"
 },
 "dependencies": {
 "express": "⁴.17.1"
 }
}
Step 2: Dockerize the application
Create a Dockerfile in your project directory with the following content:

Dockerfile

FROM node:14
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD [ "npm", "start" ]
Step 3: Define the docker-compose.yml file
Create a docker-compose.yml file in your project directory with the following content:

docker-compose.yml

version: '3'

services:
  app:
    image: your-dockerhub-username/cddemo:latest
    container_name: cddemo
    restart: always
    ports:
      - "80:3000"
    environment:
      NODE_ENV: production
Step 4: Create the GitHub Actions workflow
Create a new folder called .github and inside it, create another folder called workflows. In the workflows folder, create a file called ec2deploy.yaml with the following content:

ec2deploy.yaml

name: Build on DockerHub and Deploy to AWS
on:
  push:
    branches:
      - main
env:
  DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
  DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
  AWS_PRIVATE_KEY: ${{ secrets.AWS_PRIVATE_KEY }}
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      - name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build and push Docker image
        uses: docker/build-push-action@v2
        with:
          context: ./
          push: true
          dockerfile: ./Dockerfile
          tags: your-dockerhub-username/cddemo:latest
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v2
    - name: Login to Docker Hub
      uses: docker/login-action@v1
      with:
        username: ${{ env.DOCKERHUB_USERNAME }}
        password: ${{ env.DOCKERHUB_TOKEN }}
    - name: Set permissions for private key
      run: |
        echo "${{ env.AWS_PRIVATE_KEY }}" > key.pem
        chmod 600 key.pem
    - name: Pull Docker image
      run: |
        ssh -o StrictHostKeyChecking=no -i key.pem ubuntu@your-ec2-instance-ip 'sudo docker pull your-dockerhub-username/cddemo:latest'
    - name: Stop running container
      run: |
        ssh -o StrictHostKeyChecking=no -i key.pem ubuntu@your-ec2-instance-ip 'sudo docker stop cddemo || true'
        ssh -o StrictHostKeyChecking=no -i key.pem ubuntu@your-ec2-instance-ip 'sudo docker rm cddemo || true'
    - name: Run new container
      run: |
        ssh -o StrictHostKeyChecking=no -i key.pem ubuntu@your-ec2-instance-ip 'sudo docker run -d --name cddemo -p 80:3000 your-dockerhub-username/cddemo:latest'
Breaking down the Deployment Process
In this section, we will break down the `ec2deploy.yaml` file into smaller pieces and explain each part in detail so that anyone can understand the purpose and functionality of each section.

1. Workflow name and trigger event:

name: Build on DockerHub and Deploy to AWS
on:
 push:
 branches:
 - main
This section defines the name of the GitHub Actions workflow and specifies the trigger event. In this case, the workflow will be triggered whenever there’s a push event on the `main` branch.

2. Environment variables:

env:
 DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
 DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
 AWS_PRIVATE_KEY: ${{ secrets.AWS_PRIVATE_KEY }}
Here, we define three environment variables `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, and `AWS_PRIVATE_KEY`. These variables are fetched from the GitHub repository’s secrets which are encrypted and can be securely used in the workflow.

3. Build job:

jobs:
 build:
 runs-on: ubuntu-latest
 steps:
 - name: Checkout code
 uses: actions/checkout@v2
The `build` job is set to run on the latest version of Ubuntu. The first step in this job is to checkout the source code using the `actions/checkout@v2` action.

4. Set up Docker Buildx:

- name: Set up Docker Buildx
 uses: docker/setup-buildx-action@v1
This step sets up Docker Buildx which is an extension to the Docker CLI that enables building images using the BuildKit engine. It’s used to build and push the Docker image.

5. Login to DockerHub:

- name: Login to DockerHub
 uses: docker/login-action@v1
 with:
 username: ${{ secrets.DOCKERHUB_USERNAME }}
 password: ${{ secrets.DOCKERHUB_TOKEN }}
This step logs into DockerHub using the `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` environment variables that we defined earlier.

6. Build and push Docker image:

- name: Build and push Docker image
 uses: docker/build-push-action@v2
 with:
 context: ./
 push: true
 dockerfile: ./Dockerfile
 tags: your-dockerhub-username/cddemo:latest
In this step, we use the `docker/build-push-action@v2` action to build the Docker image using the Dockerfile in the current directory and push it to DockerHub with the specified tag.

7. Deploy job:

deploy:
 needs: build
 runs-on: ubuntu-latest
The `deploy` job is set to run on the latest version of Ubuntu and depends on the successful completion of the `build` job using the `needs` keyword.

8. Checkout code and login to Docker Hub:

steps:
 - name: Checkout code
 uses: actions/checkout@v2
 - name: Login to Docker Hub
 uses: docker/login-action@v1
 with:
 username: ${{ env.DOCKERHUB_USERNAME }}
 password: ${{ env.DOCKERHUB_TOKEN }}
These steps are similar to the ones in the `build` job. We checkout the source code and log into DockerHub for the `deploy` job.

9. Set permissions for private key:

- name: Set permissions for private key
 run: |
 echo "${{ env.AWS_PRIVATE_KEY }}" > key.pem
 chmod 600 key.pem
In this step, we create a file named `key.pem` containing the AWS private key and set its permissions to 600. This is necessary to securely access the EC2 instance using SSH.

10. Pull Docker image:

- name: Pull Docker image
 run: |
 ssh -o StrictHostKeyChecking=no -i key.pem ubuntu@your-ec2-instance-ip 'sudo docker pull your-dockerhub-username/cddemo:latest'
This step connects to the EC2 instance using SSH and pulls the latest Docker image from DockerHub.

11. Stop running container:

- name: Stop running container
 run: |
 ssh -o StrictHostKeyChecking=no -i key.pem ubuntu@your-ec2-instance-ip 'sudo docker stop cddemo || true'
 ssh -o StrictHostKeyChecking=no -i key.pem ubuntu@your-ec2-instance-ip 'sudo docker rm cddemo || true'
In this step, we stop and remove any running container with the name `cddemo` on the EC2 instance. The `|| true` after each command ensures that the workflow continues even if these commands fail (e.g., if there’s no running container with that name).

12. Run new container:

- name: Run new container
 run: |
 ssh -o StrictHostKeyChecking=no -i key.pem ubuntu@your-ec2-instance-ip 'sudo docker run -d - name cddemo -p 80:3000 your-dockerhub-username/cddemo:latest'
Finally, this step runs a new container using the latest Docker image and maps port 80 on the host to port 3000 on the container. With this understanding, you can now customize this workflow to fit your specific requirements.

IMPORTANT: A note on setting StrictHostKeyChecking=no
`StrictHostKeyChecking` is an SSH configuration option that determines the behavior when the authenticity of the remote server’s public key is unknown or doesn’t match the stored key in the local known_hosts file. By default, `StrictHostKeyChecking` is set to `yes`, which means that the SSH client will prompt the user to confirm whether to proceed with the connection or abort it.

In the context of a GitHub Actions workflow, setting `StrictHostKeyChecking` to `no` is used to avoid any interactive prompts or potential failures in the workflow due to unknown or changed host keys. When the value is set to `no`, the SSH client will automatically accept the remote server’s public key without any verification or user intervention, allowing the workflow to proceed without interruption.

However, it’s important to note that disabling `StrictHostKeyChecking` comes with security risks, as it makes the connection vulnerable to man-in-the-middle attacks. An attacker could potentially intercept the connection and present a spoofed server key, gaining access to sensitive information or compromising the integrity of the data being transferred.

In production environments or scenarios where security is critical, it’s recommended to use a more secure approach, such as pre-populating the known_hosts file with the remote server’s public key or using a custom SSH configuration that enforces strict host key checking while providing a means to programmatically handle key verification.

Step 5: Push the code to GitHub
Create a new repository on GitHub and push your code to the main branch.

Step 6: Add secrets to GitHub repository
Go to the “Settings” tab of your GitHub repository, click on “Secrets” in the left sidebar, and add the following secrets:

- DOCKERHUB_USERNAME: Your DockerHub username
- DOCKERHUB_TOKEN: Your DockerHub token or password
- AWS_PRIVATE_KEY: Your AWS private key (PEM format)


To obtain these keys, follow the instructions below:

- DOCKERHUB_USERNAME: This is simply your DockerHub username. If you don’t have a DockerHub account yet, sign up at [DockerHub](https://hub.docker.com/signup).
- DOCKERHUB_TOKEN: To create an access token for your DockerHub account, log in to DockerHub, go to [Account Settings](https://hub.docker.com/settings/security), and click on “New Access Token”. Provide a name for the token, and click “Create”. Make sure to copy the token, as you won’t be able to see it again.
- AWS_PRIVATE_KEY: This is the private key associated with your EC2 instance. When you create an EC2 instance in the AWS Management Console, you are prompted to create a new key pair or use an existing one. If you’ve already created an EC2 instance, you should have downloaded the private key (with a `.pem` extension) during the instance creation process. Make sure to store this file securely, as AWS won’t provide it again.

Once you have these keys, go to the “Settings” tab of your GitHub repository, click on “Secrets” in the left sidebar, and add the keys as secrets. This ensures that they are encrypted and can be securely used in the GitHub Actions workflow.

Step 7: Configure your EC2 instance
Make sure your EC2 instance has Docker installed and the security group allows inbound traffic on port 80. If you’re using a Windows machine, you may need to run `chmod 400 /path/to/key.pem` to set the correct permissions for your private key.


Pro Tips:
- Use a consistent naming convention for your branches, tags, and Docker images to keep your deployment pipeline clean and organized.
— Implement a robust testing strategy to ensure that your code is thoroughly tested before it’s deployed. This can include unit tests, integration tests, and end-to-end tests.
— Configure your GitHub Actions workflow to run on pull requests or specific branches, so you can test and validate changes before merging them into the main branch.
— Make use of GitHub environment protection rules and manual approvals for deploying to sensitive environments like staging or production.
— Monitor your deployments and applications using tools like AWS CloudWatch, Grafana, or Prometheus to gather insights and detect issues early.
— Regularly review and update your deployment pipeline to incorporate best practices, security patches, and new features.
— Document your deployment process and pipeline, so that your team members can easily understand and contribute to it.

Deployment /Delivery, isn’t all the same?
Continuous Delivery and Continuous Deployment are two closely related practices in the software development process, both aiming to improve the speed and reliability of releasing software updates. While they share similarities, they have some key differences in terms of their approach and level of automation.

Continuous Delivery:
Continuous Delivery is an approach that ensures your software is always in a releasable state, meaning it has passed all necessary tests and is ready to be deployed to production at any time. The main goal is to make the release process reliable, predictable, and low-risk, allowing for frequent releases with minimal manual intervention. In Continuous Delivery, the deployment to production is typically a manual step, triggered by a decision from the development team or other stakeholders.

Key aspects of Continuous Delivery include:
1. Automated testing: Ensuring that the application is thoroughly tested at every stage of the development process, minimizing the risk of shipping faulty code.
2. Version control: Keeping track of changes to the source code and managing different versions of the application.
3. Deployment automation: Automating the deployment process to various environments, such as staging or production, reducing human error and time spent on manual tasks.
4. Monitoring and feedback: Collecting feedback from users and monitoring application performance to identify issues and areas for improvement.

Continuous Deployment:
Continuous Deployment builds upon the principles of Continuous Delivery but takes automation one step further. In Continuous Deployment, every change that passes all tests and meets the criteria for a production release is automatically deployed to production, without any manual intervention. This approach results in an even faster release cycle and ensures that new features and bug fixes reach users as soon as they are ready.

Key aspects of Continuous Deployment include:
1. Fully automated pipeline: The entire process, from code commit to production deployment, is automated, requiring no manual intervention.
2. High level of confidence in testing: The application must have a comprehensive suite of tests that can reliably catch any issues before they reach production.
3. Advanced monitoring and rollback capabilities: Due to the highly automated nature of Continuous Deployment, it’s essential to have robust monitoring and rollback mechanisms in place to quickly detect and resolve any issues that might arise in production.

Continuous Delivery focuses on keeping the software in a releasable state and automating the deployment process up to the point of production, while Continuous Deployment takes it a step further by automating the final deployment to production as well. Both practices aim to improve the speed, reliability, and overall quality of software releases. The choice between Continuous Delivery and Continuous Deployment depends on the specific needs and risk tolerance of your organization.


A successful Build and Deploy
Conclusion:
Now, whenever you push changes to the main branch of your GitHub repository, the GitHub Actions workflow will build and push the Docker image to DockerHub and deploy it to your EC2 instance. This tutorial demonstrated a basic Continuous Deployment process using GitHub Actions, DockerHub, and AWS EC2. Stay tuned for more advanced topics like Testing, Elastic Beanstalk and Terraform/Ansible in future articles!
