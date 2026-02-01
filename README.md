# Vulnerable Java App - DevSecOps Demo

## Description
Demo app with several vulnerabilities for DevSecOps training.<br>
The project includes general CI/CD steps with security tools.<br>

## Some vulnerabities
1. SQL Injection<br>
2. XSS (Cross-Site Scripting)<br>
3. Hardcoded credentials<br>
4. Missing input validation<br>
5. Information disclosure<br>
and others...<br>

## How to setup the demoset (Linux)
You need 2 Linux boxes (Ubuntu) - one for Jenkins and "demo environment", another one - for docker repo

### For both servers
Check partition size (setup > 50 Gb if necessary)<br>
`df -h`<br>

Install Docker<br>
According [instructions](https://docs.docker.com/engine/install/)<br>

### For the repo server
Setup local repo for Docker:<br>
`sudo docker run -d -p 5000:5000 --restart=always --name local-registry registry:2`<br>

### For the "demo env" server
Install Java 21<br>
`sudo apt install openjdk-21-jdk`<br>

Setup the local repo as insecure storage:<br>
`echo '{"insecure-registries": ["192.168.0.5:5000"]}' | sudo tee /etc/docker/daemon.json`<br>
`sudo systemctl restart docker`<br>
(replace the "192.168.0.5" address with your repo IP)<br>

We will need the "mysql:5.7" container, tagged with the local registry IP and stored in the local registry (just for our vuln app deployment scenario). Replace "192.168.0.5" with your local repo IP<br>
`sudo docker pull mysql:5.7`<br>
`sudo docker tag mysql:5.7 192.168.0.5:5000/mysql:5.7`<br>
`sudo docker push 192.168.0.5:5000/mysql:5.7`<br>
<br>
Check that the mysql image pushed successfully:<br>
`curl -s http://192.168.0.5:5000/v2/_catalog`<br>
You should see something like this: `{"repositories":["mysql"]}`<br>
<br>
`curl -s http://192.168.0.5:5000/v2/mysql/tags/list`<br>
Desired output: `{"name":"mysql","tags":["5.7"]}`<br>

Remove the mysql images from the "demo env" server using the command:<br>
`sudo docker rmi mysql:5.7 192.168.0.5:5000/mysql:5.7`<br>

Install Cosign locally (the only purpose - use the tool for pub/private key pair generation) - [instructions](https://docs.sigstore.dev/cosign/system_config/installation/#with-the-cosign-binary-or-rpmdpkg-package)<br>
`curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64"`<br>
`sudo mv cosign-linux-amd64 /usr/local/bin/cosign`<br>
`sudo chmod +x /usr/local/bin/cosign`<br>

Generate the key pair with Cosign (for the test CI/CD please specify the password for the private key and remember it - we will use it in this scenario. In production may be decided to use or not of such password according the desired security policies)<br>
`sudo cosign generate-key-pair`<br>
After the command, you should have two files in the current path: `cosign.key` (private encrypted key) and `cosign.pub` (public key)<br>

Install Jenkins<br>
According [instructions](https://www.jenkins.io/doc/book/installing/)<br>

Add user jenkins to the docker group<br>
`sudo usermod -a -G docker jenkins`<br>
then restart server to apply the changes<br>

### Setup Jenkins configuration<br>
- Unblock Jenkins with https://192.168.0.4:8080 (replace the "192.168.0.4" address with the "demo env" server IP)<br> 
`sudo cat /var/lib/jenkins/secrets/initialAdminPassword`<br>
- Install suggested plugins<br>
- Add the "Docker Pipeline" plugin<br>
<br>
Create Pipeline:<br>
<br>
- Select "New Item" - "Pipeline"<br>
- Set "Name" ("Demo" or something)<br>
- Set "GitHub project" with an URL to the project (like "https://github.com/trainer04/DevSecOps-demo02ng.git")<br>
- Set "Definition" with "Pipeline script from SCM"<br>
- Set "SCM" as "Git"<br>
- Set "Repository URL" with your URL (like "https://github.com/trainer04/DevSecOps-demo02ng.git")<br>
- Set credentials if necessary (for private repos)<br>
- Set "Branch Specifier" as "*/main"<br>
- Set "Script Path" with the path to the Jenkins file (like "Jenkinsfile")<br>
<br>
Save the pipeline<br>

### Pipeline proxy settings<br>
To speed-up some network-related operations, the pipene includes proxy configuration, which is saved in Jenkins credentials store. If you do not use any proxies, just comment or remove the related parameters from the `Jenkinsfile`, like:<br>
`...PROXY_FOR_TOOLS = credentials('proxy-settings')...`<br>
and its usage like:<br>
`...-e HTTP_PROXY="${PROXY_FOR_TOOLS}"...`<br>