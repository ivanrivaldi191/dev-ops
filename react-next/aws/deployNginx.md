# React/NextJs + AWS
On this part we will demonstrate monorepo architecture, if you develop the single repo architecture just ignore the multiple part configurations.

## Setup EC2
- Create an EC2 instance.
	- Save .pem file [NOTE: do not remove it unless you know what you are doing].
- Create Elastic IP.
	- Save as amazon pool's IPv4 address
	- Associate it with the EC2 instance.

## Setup EC2 Instance
Connect to the EC2 instance.
- ssh -i "your_file.pem" [username]@[endpoint]
	example:
	<br>
	```
	ssh -i path/to/secret.pem ubuntu@ec2-instance.amazon.com
	```

- Install required packages.
    <br>
    For this guide we will use nvm to manage our node version and pnpm as our package manager.
    ```
    sudo apt update
    sudo apt install build-essential
    sudo apt install nginx
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
    curl -fsSL https://get.pnpm.io/install.sh | sh -
    sudo apt install git
    ```

## (Optional) Setting Up ELB instead of Elastic IP
- Create Target Group
	- Instance
	- Named it example "BE Dev Target Group"
	- HTTP Port 80, IPv4, own VPC and HTTP protocol (usually it's HTTP1)
	- Create and Save for BE and/or FE
- Create Load Balancer(LB)
	- Choose your desired LB for this subject I will used App LB
		- Insert name, example "MyApp Load Balancer"
		- Internet Facing (I will use my own domain)
		- IPv4
	- Choose the EC2 Instance VPC
	- Choose all available zones
	- Create or Choose the security group for the ELB, example:
		- Create Security Group named "ELB Sec Group"
		- Inbound HTTP and HTTPS
		- Outbound All Traffic
	- Listen to Port 80
		- On default Port 80, Listen to BE Target Group
		- Add new Rule for FE
			- Add Rule and named it, example "FE LB"
			- Add Condition "Host Header" if you have already own a domain, example "sub1.domain.com"
			- Forward it to FE Target Group
		- IF you have already own a domain and wanted to redirect to HTTPS then:
			- Redirect to URL
			- HTTPS port 443
			- Status Code 301
	- Add tags if needed
- Before you register your subdomain/domain to the ELB IP it's recommended that you create a SSL Certificate for the LB first, for this time I will use ACM(AWS Certificate Manager)
	- Create/Request Public Certificate
	- Input your domain
	- DNS validation
	- Choose your key algorithm
	- Save it and wait for around 5 to 10 minutes until it's accepted
	- Copy the CNAME name and CNAME value
- Go to your domain management
	- Add CNAME Record; with HOST is the CNAME name and VALUE is the CNAME value [This is for the certificate]
	- Add CNAME Record; with HOST is the subdomain/domain name and VALUE is the ELB DNS value [This is for the BE and FE]
	- NOTE: You won't have to set nginx config for 443 just port 80 since the 443 will be handled by the ELB

## Setup Node
```
nvm install --lts # If you have specific version use that instead otherwise this will install the lts version
```

## Setup NextJs Project (w/o CICD)
```
[Either clone the repo or create a new project]
git clone repo
cd path/to/repo
pnpm install

pnpm i -g pm2
```
## Setup NGINX
```
sudo nano /etc/nginx/sites-available/next
```

On the nginx config file
```
server {
	listen 80;
	server_name app1.domain.com;

	charset utf-8;

	location / {
		proxy_pass http://127.0.0.1:3000;
		proxy_http_version 1.1;
		proxy_set_header Host $host;
		proxy_cache_bypass $http_upgrade;
	}
}

server {
	listen 80;
	server_name app2.domain.com;

	charset utf-8;

	location / {
		proxy_pass http://127.0.0.1:3001;
		proxy_http_version 1.1;
		proxy_set_header Host $host;
		proxy_cache_bypass $http_upgrade;
	}
}
```

Link the config file
```
sudo ln -s /etc/nginx/sites-available/next /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

## Configuring the .env files
```
cp ./apps/<app_name>/.env.example ./apps/<app_name>/.env
nano ./apps/<app_name>/.env
```

On the .env on monorepo architecture there may be varies, but here's some example
```
NEXT_PUBLIC_ROOT_API_URL=<back_end_endpoint_base>
NEXT_PUBLIC_ROOT_API_URL_MASTER=<back_end_endpoint_master>

IMAGE_HOST=<buket>.s3.<region>.amazonaws.com
IMAGE_HOST_MOCK=picsum.photos
FLAG_CDN=flagcdn.com
```

## Running the app
```
pnpm run <app_name>:build
pm2 start pnpm --name "APP_NAME" --cwd path/to/the/mono/repo/artifact -- start
```

To check if either the app is running, you can run
```
pm2 list
pm2 log <id_number>
```

And after that you can try hit the endpoint.
<br>
Don't forget to register you domain to your elastic IP address!
<br>
Right now it should be able to accessed but it's still on http

## Setup SSL
For the https part, you also have so many ways, for now I will demonstrate the nginx + certbot
```
sudo apt install certbot
sudo apt-get install python3-certbot-nginx
sudo certbot certonly --nginx -d your_domain.com
```

It will prompt you into several rules and agreement
If all goes well it should be like this

```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/your_domain.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/your_domain.com/privkey.pem
This certificate expires on 2025-10-10.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.
```

## Setup SSL NGINX Config

For port 80 you can forced it into redirect to https like this
```
location / {
	return 301 http://$host$request_uri;
	# return 301 https://$host$request_uri;
}

[or]
return 301 http://$host$request_uri;
# return 301 https://$host$request_uri;
```
Now after the certificate has been created update you nginx config into something like this
```
server {
	listen 443 ssl;

	# SSL via LetsEncrypt
	ssl_certificate <your_certificate>;
	ssl_certificate_key <your_certificate_key>;

	server_name app1.domain.com;

	# Security, example:
	# ssl_session_cache shared:SSL:20m;
	# ssl_session_timeout 60m;
	# ssl_prefer_server_ciphers on;
	# ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	# ssl_ciphers HIGH:!aNULL:!MD5;
	# ssl_dhparam /etc/ssl/certs/dhparams.pem;

	#access_log /var/log/your_domain.com.access.log;
	error_log /var/log/your_domain.com.error.log warn;


	# Redirect https request of this exception URL to http
	# location /except {
	#     return 301 http://$host$request_uri;
	# }

    charset utf-8;

	location / {
		proxy_pass http://127.0.0.1:3000;
		proxy_http_version 1.1;
		proxy_set_header Host $host;
		proxy_cache_bypass $http_upgrade;
	}
}

server {
	listen 443 ssl;

	# SSL via LetsEncrypt
	ssl_certificate <your_certificate>;
	ssl_certificate_key <your_certificate_key>;

	server_name app2.domain.com;

	# Security, example:
	# ssl_session_cache shared:SSL:20m;
	# ssl_session_timeout 60m;
	# ssl_prefer_server_ciphers on;
	# ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	# ssl_ciphers HIGH:!aNULL:!MD5;
	# ssl_dhparam /etc/ssl/certs/dhparams.pem;

	#access_log /var/log/your_domain.com.access.log;
	error_log /var/log/your_domain.com.error.log warn;


	# Redirect https request of this exception URL to http
	# location /except {
	#     return 301 http://$host$request_uri;
	# }

    charset utf-8;

	location / {
		proxy_pass http://127.0.0.1:3001;
		proxy_http_version 1.1;
		proxy_set_header Host $host;
		proxy_cache_bypass $http_upgrade;
	}
}
```

## Setup crontab/cronjob to renew SSL certificate
If you are lazy to remember everything regarding the certbot just make an crontab for it
```
sudo crontab -e

0 0 * * * certbot renew --nginx --quiet
```

## Common Issues
### NVM and PNPM does not recognized on first installation
- If this happened, try to exit the instance then login into the same instance again, most likely it hasn't been loaded into ubuntu/current user.

### Restart NGINX
```
sudo systemctl restart nginx
```

### Removing Certificates
```
sudo certbot certificates
sudo certbot delete
```

<br>
Study References:
<br>

- [Snippet Commands SSL Certificate](https://medium.com/@vinoji2005/guide-to-setup-lets-encrypt-ssl-in-nginx-be3d641bb58a)
<br>

- [Snippet Config](https://gist.github.com/dorelljames/9da3063878b9c3030d6538b6724122ac)
<br>

- [Snippet Config Microfrontend](https://medium.com/@djdevesh524/nginx-setup-for-your-first-micro-frontend-application-b0c1179cefa6)
<br>

- [Youtube to Visualize](https://youtu.be/vtlZWotRmTI?si=wYDHcT5EPxRBI3-7)
<br>
