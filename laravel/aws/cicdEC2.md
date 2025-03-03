# CI/CD CodeDeploy + CodePipeline
Be wise, since this is just a guide on how to create basic CI/CD if you wanted to custom do kindly research on your own.

## AMI disk image
Before you proceed further you might have the idea of wanting to back up the instance into an image, here's how:
- Right Click EC2 Instance > Image > Create Image
- Insert the image name and description
- Create an Image

## IAM Role Setup
It's recommended to create IAM Role for CodeDeploy and CodePipeline
<br>
EC2 Code Deploy Role
1. Go to IAM in AWS
2. Create Role
    - AWS Service
    - EC2
    - Go Next.
3. Add Permission "AmazonEC2RoleforAWSCodeDeploy"
4. Insert the Role Name, example: "EC2CodeDeployRole"
5. Create the Role.

Code Deployment Role
1. EC2 Code Deploy Role
2. Go to IAM in AWS
3. Create Role
    - AWS Service
    - Choose "CodeDeploy" in the dropdown [Use cases for other AWS service]
    - Go Next.
3. The Permission will be pre selected, if the choice is empty search for "AWSCodeDeployRole"
4. Insert the Role Name, example: "CodeDeployRole"
5. Create the Role.

## Setup Role to EC2
I assume you have already created an EC2 instance, if you haven't you migh wanted to check <a href="deployNginx.md">How to deploy Laravel to AWS EC2 using NGINX.</a>
<br>
If you have created an instance or in process on creating you can assign IAM instance profile / IAM Role and choose your created EC2 Code Deploy Role.

## Setup EC2 CodeDeploy Agent
Now that you have created an EC2 instance you have 2 options when creating the CodeDeploy Agent:
- While on the process of creating EC2 instance there will be a field named User Data, you can init the command there, or
- Connect to you EC2 instance and insert the command
```
sudo apt update
sudo apt install ruby-full
sudo apt install wget

cd /home/ubuntu
wget https://bucket-name.s3.region-identifier.amazonaws.com/latest/install

chmod +x ./install
sudo ./install auto

systemctl status codedeploy-agent
```

I won't go through all the commands since the documentation explained perfectly what does it do, please refer to the Study References.

## Setup CodeDeploy
- Create an Application name, example: "MyApp-CICD"
- Choose EC2/on-premises on the Compute platform
- After that create deployment group
    - Insert the deployment group name, example: "MyApp-CICD-DPL"
    - Choose the service role which you have created from the start, example: "CodeDeployRole"
    - Deployment Type of in-place (if you know what you are doing choose the others if you are here to learn simply choose in-place first as the default)
    - Environment Configuration AWS EC2 instances
    - Key: name | Value: your instance name
    - Deployment Settings will be AllAtOnce
    - If your application is just single application do not use load balancer.

## Setup CodePipeline
- Create a Pipeline name, example: "MyApp-CICD-PIPELINE"
- Next, choose your own source provider example: GitHub. Connect it into your repository
- Choose the default branch that will be the target of CI/CD
- Skip the build and test if you don't need it
- On the deploy stage choose AWS CodeDeploya and your own region the application name will be the created CodeDeploy Application Name, example: "MyApp-CICD". So does on the deployment group.
- Finish.

## Setup the script
On your repository create a file named `appspec.yml` on root level directory. For this example I will used root as the user.
<br>
Here is the example/template:
```
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/path/to/your/laravel
hooks:
  BeforeInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: scripts/deploy_laravel.sh
      timeout: 300
      runas: root
  ApplicationSstart:
    - location: scripts/start_server.sh
      timeout: 120
      runas: root
  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 120
      runas: root
```

After that create a directory named `scripts`, in this directory will consists total of 4 files .sh from the example above with the content of each type as of following:
- install_dependencies.sh
```
#!/bin/bash
sudo apt install nginx
```

- deploy_laravel.sh
```
#!/bin/bash

export COMPOSER_ALLOW_SUPERUSER=1 # This is required if you used root/superuser
composer install -d /var/www/laravel/paketur-backend
php /var/www/laravel/paketur-backend/artisan migrate

# Clear Previous Caches
php /var/www/laravel/paketur-backend/artisan config:clear
php /var/www/laravel/paketur-backend/artisan cache:clear
php /var/www/laravel/paketur-backend/artisan route:clear


# Optimize Laravel
php /var/www/laravel/paketur-backend/artisan optimize
```

- start_server.sh
```
#!/bin/bash
systemctl start nginx
```

- stop_server.sh
```
#!/bin/bash
systemctl stop nginx
```

After that the CI/CD should be done.

## Common Issues
### User <script_runas_user> failed with exit code 127
Usually this error happened because of the repository access, you can try to check it by running
```
ls -la
ls -ltr
```

After that. you have choices such as chown or chmod

### User <script_runas_user> failed with exit code 127, even after the first solution
This might be interesting, but if you still encounter the same issue please do read the error log.
<br>
The location is usually at:
<br>
CodePipeline > AWS CodeDeploy > View Events > [Usually there will be a log/file why It won't work, example] ScriptFailed

### Yum does not exists
This one should be easy, check for your OS if it's the basic Ubuntu, usually they don't have that so the option is to use sudo apt

Study References:
<br>

- [AWS Documentation](https://docs.aws.amazon.com/codedeploy/latest/userguide/codedeploy-agent-operations.html)
<br>

- [Snippet Command Steps](https://github.com/pietheinstrengholt/aws-codepipeline-laravel/blob/master/README.md)
<br>

- [Snippet Command Steps](https://github.com/aarhemareddy/aws_cicd_pipline_codedeploy/blob/main/README.md)
<br>

- [Youtube to Visualize](https://youtu.be/531i-n5FMRY?si=yxFXzHmw4LBdO_hr)
<br>
