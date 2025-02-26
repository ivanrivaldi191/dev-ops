# Laravel + AWS + RDS Setup
1. Create an EC2 instance.
	- Save .pem file [NOTE: do not remove it unless you know what you are doing].
2. Create Elastic IP.
	- Save as amazon pool's IPv4 address
	- Associate it with the EC2 instance.
3. Create a Security Group for the EC2 & RDS instance.
	- Add inbound rules of the RDS (example: PostgreSQL or MySQL); the source would be anywhere.
4. Create RDS.
	- You have two option here:
		- Either you can set it publicly or,
		- Set it so it can only be accessed through the EC2 instance.
	Note: If you set it publicly, you have an issue regarding safety and of course, you can still use it to connect to the EC2 instance manually.
5. Connect to the EC2 instance.
	- ssh -i "your_file.pem" [username]@[endpoint]
		example:
		<br>
		```
		ssh -i path/to/secret.pem ubuntu@ec2-instance.amazon.com
		```
	- Now you have multiple options here; either you want to use Nginx or Apache Web Server. As for now, we will use Nginx.
		<br>
		The part on ppa:ondrej/php is necessary since the base Linux will have a hard time fetching the package from PHP. If you use Debian, you might want to check if it's still ppa:ondrej or the other. The main search keyword is "Ondrej Sury PHP package linux distro."
		```
		sudo add-apt-repository ppa:ondrej/php
		sudo apt update
		sudo apt install nginx
		sudo apt install php php-fpm php-mysql php-mbstring php-xml php-curl php-zip php-pgsql
		sudo apt install composer
		sudo nano /etc/nginx/sites-available/laravel
		```

		On the nginx config file
		```
		server {
		    listen 80;
		    server_name your_domain.com;
		    root /var/www/path/to/laravel/public;

		    index index.php index.html index.htm;

		    charset utf-8;

		    location / {
		        try_files $uri $uri/ /index.php?$query_string;
		    }

		    location ~ \.php$ {
		        include fastcgi_params;
		        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name; # note: always check for your own fastcgi_params location if it does not exists use fastcgi.confd instead
		        fastcgi_pass unix:/var/run/php/php[version]-fpm.sock;
		    }

		    location ~ /\.ht {
		        deny all;
		    }
		}
		```

		Link the config file
		```
		sudo ln -s /etc/nginx/sites-available/laravel /etc/nginx/sites-enabled/
		sudo systemctl restart nginx
		```

		Optional/Mandatory Part
		If you wanted to easily access /var/www, then you might want to change your access using chown and chmod.
		example:
		```
		sudo chown -R ec2-user:apache /var/www
		sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
		find /var/www -type f -exec sudo chmod 0664 {} \;
		```

		After that you can either clone your repository or create your project at /var/www/
		example: 
		```
		cd /var/www
		git clone https://<PAT>@github.com/<username_or_org>/<repo>.git

		cd ./<repo>
		composer install

		[if it's conflicted you can use]
		composer install --ignore-platform-req=php

		[but it's not recommended since it's a workaround not a solution]

		cp .env.example .env

		php artisan key:generate
		php artisan jwt:secret

		[you can also run your/others own vendor secret, example]
		php artisan ciphersweet:generate-key

		chmod -R 777 ./storage/

		nano .env
		```

		on the env file things that you might want to change is mostly on database and cloud storage, example
		```
		DB_CONNECTION=pgsql
		DB_HOST=your_rds_endpoint_host
		DB_PORT=5432 # this is the default, you can change it into your port
		DB_DATABASE=your_database_name
		DB_USERNAME=your_username
		DB_PASSWORD=your_password

		AWS_ACCESS_KEY_ID=YourAwsKeyId
		AWS_SECRET_ACCESS_KEY=YourAwsSecretKey
		AWS_DEFAULT_REGION=ap-southeast-1
		AWS_BUCKET=YourAwsBucket
		```

		If you wanted to access your RDS from EC2 just install the datbase client, example:
		```
		sudo apt install postgresql-client-common
		sudo apt install postgresql-client

		psql -h <rds_endpoint> -p <port> -U <username>
		```

		After you setup the database don't forget to migrate and/or seed
		```
		php artisan migrate
		php artisan db:seed
		```

		Don't forget to register you domain to your elastic IP address!
		Right now it should be able to accessed but it's still on http

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

		Now after the certificate has been created update you nginx config into something like this
		```
		server {
		    listen 443 ssl;

		    # SSL via LetsEncrypt
		    ssl_certificate <your_certificate>;
		    ssl_certificate_key <your_certificate_key>;

		    server_name your_domain.com;

		    # Security, example:
		    # ssl_session_cache shared:SSL:20m;
		    # ssl_session_timeout 60m;
		    # ssl_prefer_server_ciphers on;
		    # ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
		    # ssl_ciphers HIGH:!aNULL:!MD5;
		    # ssl_dhparam /etc/ssl/certs/dhparams.pem;

		    #access_log /var/log/your_domain.com.access.log;
		    error_log /var/log/your_domain.com.error.log warn;

		    root /var/www/path/to/laravel/public;
		    index index.php;

		    # Just process requests normally
		    location / {
		        try_files $uri $uri/ /index.php?$query_string;
		    }

		    # Redirect https request of this exception URL to http
		    # location /except {
		    #     return 301 http://example.com$request_uri;
		    # }

		    location ~ \.php$ {
		        include fastcgi_params;
		        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
		        fastcgi_pass unix:/var/run/php/php-fpm.sock;
		    }
		}
		```

		If you are lazy to remember everything regarding the certbot just make an crontab for it
		```
		sudo crontab -e

		0 0 * * * certbot renew --nginx --quiet
		```
<br>
Study References:
<br>
[Procedural Steps](https://dev.to/mdarifulhaque/step-by-step-deploy-laravel-app-to-cloud-aws-google-azure-digitalocean-with-cicd-using-github-actionsgitlab-ci-2hb9)
<br>
[Procedural Steps](https://medium.com/@bjnandi/how-to-deploy-laravel-application-to-ec2-instance-using-github-action-automatically-ssh-deploy-e198b0a3136d)
<br>
[Snippet Commands](https://medium.com/@vinoji2005/guide-to-setup-lets-encrypt-ssl-in-nginx-be3d641bb58a)
<br>
[Snippet Config](https://gist.github.com/dorelljames/9da3063878b9c3030d6538b6724122ac)
<br>
[Snippet Commands](https://github.com/hardcoreprogrammingwarrior/deploy-laravel-8-with-ec2-linux-2-and-rds)
<br>
[Youtube to Visualize](https://youtu.be/mBEdFlw4ybc?si=_q0iN_7ANfnfCClW)
<br>
