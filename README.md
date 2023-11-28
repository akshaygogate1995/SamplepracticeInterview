# SamplepracticeInterview

#DockerFile

# Use an official OpenJDK runtime as a base image
FROM openjdk:11-jre-slim

# Set the working directory in the container
WORKDIR /app

# Copy the project files into the container
COPY . /app

# Install Maven
RUN apt-get update && \
    apt-get install -y maven && \
    rm -rf /var/lib/apt/lists/*

# Run Maven build
RUN mvn clean install

# Command to run the application (adjust as needed)
CMD ["java", "-jar", "target/your-application.jar"]

# Production DOckerFile

# Use a minimal base image for production
FROM adoptopenjdk:11-jre-hotspot-bionic

# Set the working directory in the container
WORKDIR /app

# Copy only the necessary files for dependency resolution
COPY ./pom.xml ./pom.xml

# Copy the project files and build
COPY ./src ./src
RUN mvn clean install -DskipTests

# Remove Maven artifacts and source code to reduce image size
RUN rm -rf ./src ./target

# Specify the JAR file to run
ARG JAR_FILE=target/your-application.jar

# Copy the application JAR into the container
COPY ${JAR_FILE} app.jar

# Expose the port the application runs on
EXPOSE 8080

# Define environment variables (adjust as needed)
ENV SPRING_PROFILES_ACTIVE=production

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]

# Ansible Playbooks

1. Java Application Deployment

---
- name: Deploy Java Application
  hosts: your_production_servers
  become: true
  tasks:
    - name: Update apt package cache
      apt:
        update_cache: yes

    - name: Install OpenJDK
      apt:
        name: openjdk-11-jre-headless
        state: present

    - name: Copy application JAR file
      copy:
        src: /path/to/your-application.jar
        dest: /opt/your-application.jar

    - name: Set up systemd service for the application
      systemd:
        name: your-application
        enabled: yes
        state: started
      notify:
        - Reload Nginx
  handlers:
    - name: Reload Nginx
      systemd:
        name: nginx
        state: restarted


2. Configure Nginx as Reverse Proxy Playbook:

---
- name: Configure Nginx as Reverse Proxy
  hosts: your_nginx_servers
  become: true

  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Configure Nginx as reverse proxy
      template:
        src: nginx-reverse-proxy.conf.j2
        dest: /etc/nginx/sites-available/your-application
      notify:
        - Reload Nginx

    - name: Enable the Nginx site
      file:
        src: /etc/nginx/sites-available/your-application
        dest: /etc/nginx/sites-enabled/your-application
        state: link

  handlers:
    - name: Reload Nginx
      systemd:
        name: nginx
        state: reloaded

      
3. Server Hardening Playbook:

---
- name: Server Hardening
  hosts: your_production_servers
  become: true

  tasks:
    - name: Update apt package cache and upgrade packages
      apt:
        update_cache: yes
        upgrade: dist

    - name: Install essential packages
      apt:
        name: "{{ item }}"
        state: present
      loop:
        - fail2ban
        - unattended-upgrades
        - ufw

    - name: Configure Unattended Upgrades
      copy:
        src: 10periodic
        dest: /etc/apt/apt.conf.d/10periodic
      notify:
        - Restart Unattended Upgrades

    - name: Configure UFW (Uncomplicated Firewall)
      ufw:
        rule: allow
        port: "{{ item }}"
        comment: "Allow access to port {{ item }}"
        state: enabled
      loop:
        - 22  # SSH
        - 80  # HTTP
        - 443 # HTTPS
      notify:
        - Restart UFW

    - name: Secure SSH configuration
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
        state: present
      loop:
        - { regexp: '^PermitRootLogin', line: 'PermitRootLogin no' }
        - { regexp: '^PasswordAuthentication', line: 'PasswordAuthentication no' }
        - { regexp: '^PermitEmptyPasswords', line: 'PermitEmptyPasswords no' }
      notify:
        - Restart SSH

  handlers:
    - name: Restart Unattended Upgrades
      systemd:
        name: unattended-upgrades
        state: restarted

    - name: Restart UFW
      command: ufw reload

    - name: Restart SSH
      systemd:
        name: ssh
        state: restarted

#Shell Scripts

1. Server Resource Monitoring Script:

#!/bin/bash

# Server Resource Monitoring Script

# Set variables
LOG_FILE="/var/log/resource_monitor.log"

# Log current date and server resource usage to a file
echo "Date: $(date)" >> "$LOG_FILE"
echo "Uptime: $(uptime)" >> "$LOG_FILE"
echo "Memory Usage: $(free -h)" >> "$LOG_FILE"
echo "Disk Usage: $(df -h)" >> "$LOG_FILE"
echo "---------------------------------------------" >> "$LOG_FILE"

2. Database Backup Script:

#!/bin/bash

# Database Backup Script

# Set variables
DB_USER="your_db_user"
DB_PASSWORD="your_db_password"
DB_NAME="your_database"
BACKUP_DIR="/path/to/backups"
DATE=$(date +"%Y%m%d%H%M%S")
BACKUP_FILE="$DB_NAME-$DATE.sql"

# Create backup directory if it doesn't exist
mkdir -p "$BACKUP_DIR"

# Dump the database
mysqldump -u "$DB_USER" -p"$DB_PASSWORD" "$DB_NAME" > "$BACKUP_DIR/$BACKUP_FILE"

# Compress the backup file
gzip "$BACKUP_DIR/$BACKUP_FILE"

# Rotate backups to keep only the last 7 days
find "$BACKUP_DIR" -name "$DB_NAME*.gz" -type f -mtime +7 -exec rm {} \;

# Jenkinsfile

pipeline {
    agent {
        docker {
            image 'maven:3.8.4-jdk-11'
            label 'docker-agent'
        }
    }

    environment {
        DOCKER_IMAGE = "your-docker-image:latest"
        K8S_NAMESPACE = "your-kubernetes-namespace"
        K8S_DEPLOYMENT = "your-kubernetes-deployment"
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    checkout scm
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    sh 'mvn clean package'
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    sh 'mvn test'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t $DOCKER_IMAGE .'
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh "docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD"
                        sh "docker push $DOCKER_IMAGE"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'kube-config-credentials', serverUrl: 'your-kubernetes-api-server']) {
                        sh "kubectl set image deployment/$K8S_DEPLOYMENT $K8S_DEPLOYMENT=$DOCKER_IMAGE --namespace=$K8S_NAMESPACE"
                    }
                }
            }
        }
    }
}



