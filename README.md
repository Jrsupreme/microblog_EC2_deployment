# Microblog EC2 Deployment

Deployment Workload 3: Manual Deployment to EC2 with Monitoring

## Purpose

This workload marks a transition from relying on AWS managed services to independently building and managing the infrastructure. Unlike Workloads 1 and 2, which utilized AWS Elastic Beanstalk for automated provisioning, Workload 3 emphasizes a hands-on approach to manually constructing and maintaining the infrastructure for a web application.

### Steps

1. **Clone Repository:** Clone the "microblog_EC2_deployment" repository to your GitHub account.
2. **EC2 Instance Creation:**
    - Create a t3.micro Ubuntu EC2 instance named "Jenkins".
    - Install Jenkins on the instance.
    - Configure the security group to allow SSH, HTTP traffic, and ports needed for Jenkins and other services. (Also allow traffic on port 9100 for prometheus node exporter)
3. **Server Configuration:**
    - Install `python3.9`, `python3.9-venv`, `python3-pip`, and `nginx` on the server.
4. **Application Setup:**
    - Clone the repository onto the server.
    - Create and activate a virtual environment named "venv".
    - Install application dependencies and packages using `pip install -r requirements.txt` and `pip install gunicorn pymysql cryptography`.
5. **Environment Variable:**
    - Set the `FLASK_APP` environment variable to `microblog.py`.
    
    ```bash
    FLASH_APP=microblog.py
    ```
    
6. **Database Setup:**
    - Run `flask translate compile` to compile translations catalogs. These catalogs contain translations of text strings used in the application, allowing it to be displayed in different languages..
    - Run `flask db upgrade` to create or update the database schema. This command helps your Flask application keep its database structure up-to-date.
7. **Nginx Configuration:**
    - Edit the configuration file at `/etc/nginx/sites-enabled/default` and update the "location" block to:
    
    Nginx
    
    ```bash
    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;   
    }
    ```
    This Nginx configuration block is setting up a reverse proxy to forward incoming requests to our Flask application gunicorn running on port 5000.
    
8. **Application Startup:**
    - Run `gunicorn -b :5000 -w 4 microblog:app` to start the application on port 5000 with 4 worker processes. This command instructs gunicorn to, use 4 worker processes (`w 4`), and import the application object (`microblog:app`).
    - Access the application by entering the server's Public IP address into your web browser's URL bar.
    - After the application is running, stop it by pressing Ctrl+C in the terminal.
9. **Jenkins Pipeline Creation:**
- Modify the `Jenkinsfile` to define stages for build, test, deploy, clean, and OWASP FS scan.
    
    
    - Build: Install dependencies, create virtual environment, set variables, and configure databases.
    
    ```bash
     stage('Build') {
                steps {
                    sh '''#!/bin/bash
                    python3.9 -m venv venv
                    source venv/bin/activate
                    export FLASK_APP=microblog.py
                    pip install -r requirements.txt
                    flask translate compile
                    flask db upgrade
                    '''
    ```
    
    - Test: Run unit tests using pytest in the script located at `tests/unit/`.
    
    ```bash
    stage ('Test') {
                steps {
                    sh '''#!/bin/bash
                    source venv/bin/activate
                    py.test ./tests/unit/ --verbose --junit-xml test-reports/results.xml
                    '''
    ```
    
    - OWASP FS Scan: Use the OWASP Dependency-Check plugin to scan dependencies for vulnerabilities during the build stage.
    
    ```bash
     stage('OWASP FS SCAN') {
                steps {
                    dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
    ```
    
    - This stage checks the project for any known security issues in the libraries or tools you're using. It runs a scan to find out if there are any vulnerabilities in the dependencies (things like third-party packages).
    
    - Clean: Perform any cleanup tasks .
    
    ```bash
    stage ('Clean') {
                steps {
                    sh '''#!/bin/bash
                    if [[ $(ps aux | grep -i "gunicorn" | tr -s " " | head -n 1 | cut -d " " -f 2) != 0 ]]
                    then
                    ps aux | grep -i "gunicorn" | tr -s " " | head -n 1 | cut -d " " -f 2 > pid.txt
                    kill $(cat pid.txt)
                    exit 0
                    fi
                    '''
    ```
    
    - The clean stage stops any Gunicorn servers (which run the app) that are still running after the deployment. This makes sure there’s no confusion or conflict with older versions of the app still running in the background. It ensures that when you redeploy the app, only the latest version is running, preventing issues like having multiple copies running at the same time.
    
    - Deploy: Run commands necessary to deploy the application.

```bash
  stage('Deploy') {
            steps {
                sh '''#!/bin/bash
                source venv/bin/activate
                gunicorn -b :5000 -w 4 microblog:app
                '''
            }
```

1. To ensure the Gunicorn process continues running after the deployment stage, create a systemd service unit. This will manage the application's lifecycle, ensuring it starts and remains running even after the pipeline completes.
    
    **Steps:**
    
    1. **Create a systemd service unit file:** Create a file named `microblog.service` in the `/etc/systemd/system/` directory.
    2. **Add the following contents to the file**:
    
    ```bash
    [Unit]
    Description=gunicorn service to deploy microblog
    After=network.target
    
    [Service]
    User=jenkins
    Group=jenkins
    WorkingDirectory=/var/lib/jenkins/workspace/workload_3_main/
    Environment="PATH=/var/lib/jenkins/workspace/workload_3_main/venv/bin"
    ExecStart=/var/lib/jenkins/workspace/workload_3_main/venv/bin/gunicorn -w 4 -b :5000 microblog:app
    
    [Install]
    WantedBy=multi-user.target
    ```
    
    1. **Reload systemd:**
    - Run the following command to inform systemd about the newly created service file:
        
        ```bash
        sudo systemctl daemon-reload
        ```
        
    1. **Start the service:**
    - Run the following command to start the Gunicorn process as a systemd service:
    
    ```bash
    sudo systemctl start microblog
    ```
    
    1. **Check the service status:**
    - Verify that the service is running properly using the following command:
    
    ```bash
    sudo systemctl status microblog
    ```
    
    1. **Add Service Restart to Jenkins** 
    - **Modify Jenkinsfile:** So that Jenkins automatically restarts the service during deployment.
        - Add the following step to the `Deploy` stage in your Jenkinsfile:
    
    ```bash
    stage('Deploy') {
        steps {
            sh '''#!/bin/bash
            sudo systemctl restart microblog
            '''
        }
    }
    ```
    
    By following these steps and creating a systemd service, you ensure that your Gunicorn process continues running even after the Jenkins pipeline completes. This guarantees the availability of your microblog application and prevents manual intervention to restart the process.
    
2. **Jenkins Plugin Installation:**
- Install the "OWASP Dependency-Check" plugin in Jenkins. Configure it to check dependencies automatically during the build stage.
- This looks through all the libraries the project is using and checks if any of them have reported security issues. The plugin runs during the **OWASP FS SCAN** stage in the Jenkins pipeline. It scans the project after the build or test stages and before deployment.
- This plugin helps ensure the application isn't using any insecure or outdated libraries that could make it vulnerable to attacks. By catching these issues early, you can fix or update the libraries before putting the app into production, making it more secure.

1. **MultiBranch Pipeline Creation:**
    - Create and run a MultiBranch Pipeline named "workload_3" in Jenkins.
2. **Setting Up Prometheus and Grafana for Monitoring**
    
    ### Steps
    
    1. **Create an EC2 Instance:**
    - Launch a new t3.micro EC2 instance named "Monitoring".
    - Configure the security group to allow inbound traffic on port 9090 (Prometheus) and port 3000 (Grafana).
    
     **2.  Install Prometheus and Grafana:**
    
    - Refer to the official Prometheus and Grafana documentation for detailed installation instructions and system-specific configurations:
        - **Prometheus:** [https://prometheus.io/](https://prometheus.io/)
        - **Grafana:** [https://grafana.com/](https://grafana.com/)
    
    Note: (Make sure to install the prometheus node exporter in the server where the source code is)
    
    1. **Configure Prometheus:**
        - Edit the Prometheus configuration file:  /etc/prometheus/prometheus.yml/red
        - add the following:
        
        ```bash
        - job_name: "node_exporter"
            static_configs:
              - targets: ["<Code_sever's_privateIP>:9100"]
        ```
        
    2. **Configure Grafana:**
        - Access Grafana in your web browser using the server's public IP address and port 3000.
        - Create a data source for Prometheus.
        - Import dashboards or create custom dashboards to visualize metrics from your application.
   
    ![Screenshot (81)](https://github.com/user-attachments/assets/651c7e9c-dac6-420a-99d7-1a5c04857094)


### Issues and Troubleshooting

1. Instance Size
    - t3.micro instance was crashing during the OWASP scan stage.
    - Solution: Changed to a t3.medium instance to run jenkins successfully
2. Gunicorn Installation
    - Gunicorn wasn't installed before starting the environment
    - Resulted in inability to run pip or install required libraries and dependencies
3. Pip Usage Issues
    - Even after installing Gunicorn, pip couldn't be used
    - Solution:
        - Checked pip version (found to be outdated)
        - Updated pip using:
            
            ```bash
            $sudo apt update
            $pip install --upgrade pip
            ```
            
        - Installed application dependencies and libraries:
            
            ```bash
            $pip install -r requirements.txt
            $pip install gunicorn pymysql cryptography
            ```
            
4. Could not get the application to “StayAlive” after it ran through the Jenkins Pipeline.
    - Work around listed in step 10.

### Advantages of Provisioning Own Resources vs. Using Elastic Beanstalk

- **Greater control**: Full control over the infrastructure allows for customization to meet specific needs.
- **Increased flexibility**: Different EC2 instance types, operating systems, and configurations can be selected based on application requirements.
- **Cost optimization**: Resource allocation can be fine-tuned to ensure efficient use and minimize costs.

### Is the Infrastructure Created in This Workload a "Good System"?

- **Solid foundation**: The infrastructure provides a good base for deploying a production-ready application.
- **Needs further optimization**: Improvements in scalability, high availability, security, performance, and monitoring can be made.
- **Potential for customization**: The system can be tailored and optimized based on specific application needs.

### Optimizing the Infrastructure

- **Implement auto-scaling**: AWS Auto Scaling can be used to automatically adjust EC2 instances based on traffic demands.
- **Configure load balancing**: Elastic Load Balancing can distribute traffic across multiple instances to improve availability and performance.
- **Backup and recovery:** Implement a backup strategy to protect data and application state.

## Conclusion

Finishing Workload 3 was definitely tough, but also really rewarding. Moving away from using managed services and actually building the infrastructure myself gave me a real sense of how much goes into deploying an application. I had to figure out how to set up Jenkins, handle dependencies, and configure things like Nginx and Gunicorn, which required a lot of trial and error. Adding monitoring tools like Prometheus and Grafana also taught me how to keep an eye on server performance.

Compared to the first two workloads, this one was more challenging because it forced me to dive deeper into managing servers, handling network settings, and making sure everything worked together smoothly. Despite the struggles, I learned a lot about how important automation is and how to approach problems step by step. In the end, this workload really helped me understand what it takes to deploy and maintain a system on my own.
