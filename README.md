https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-with-docker-compose

Step 1 — Defining the Web Server Configuration
Before running any containers, our first step will be to define the configuration for our Nginx web server. Our configuration file will include some WordPress-specific location blocks, along with a location block to direct Let’s Encrypt verification requests to the Certbot client for automated certificate renewals.

First, create a project directory for your WordPress setup called wordpress and navigate to it:

mkdir wordpress && cd wordpress

Next, make a directory for the configuration file:

mkdir nginx-conf

Open the file with your favorite editor:

vim nginx-conf/nginx.conf

In this file, we will add a server block with directives for our server name and document root, and location blocks to direct the Certbot client’s request for certificates, PHP processing, and static asset requests.

Paste the following code into the file. Be sure to replace example.com with your own domain name:

server {
        listen 80;
        listen [::]:80;

        server_name example.com www.example.com;

        index index.php index.html index.htm;

        root /var/www/html;

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }

        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass wordpress:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $fastcgi_path_info;
        }

        location ~ /\.ht {
                deny all;
        }

        location = /favicon.ico { 
                log_not_found off; access_log off; 
        }
        location = /robots.txt { 
                log_not_found off; access_log off; allow all; 
        }
        location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
                expires max;
                log_not_found off;
        }
}

Step 2 — Defining Environment Variables
Your database and WordPress application containers will need access to certain environment variables at runtime in order for your application data to persist and be accessible to your application. These variables include both sensitive and non-sensitive information: sensitive values for your MySQL root password and application database user and password, and non-sensitive information for your application database name and host.

Rather than setting all of these values in our Docker Compose file — the main file that contains information about how our containers will run — we can set the sensitive values in an .env file and restrict its circulation. This will prevent these values from copying over to our project repositories and being exposed publicly.

In your main project directory, ~/wordpress, open a file called .env:

vim .env

The confidential values that we will set in this file include a password for our MySQL root user, and a username and password that WordPress will use to access the database.

Add the following variable names and values to the file. Remember to supply your own values here for each variable:

MYSQL_ROOT_PASSWORD=your_root_password
MYSQL_USER=your_wordpress_database_user
MYSQL_PASSWORD=your_wordpress_database_password

We have included a password for the root administrative account, as well as our preferred username and password for our application database.

Save and close the file when you are finished editing.

Because your .env file contains sensitive information, you will want to ensure that it is included in your project’s .gitignore and .dockerignore files, which tell Git and Docker what files not to copy to your Git repositories and Docker images, respectively.

If you plan to work with Git for version control, initialize your current working directory as a repository with git init:

git init
Then open a .gitignore file:

nano .gitignore
Add .env to the file:

~/wordpress/.gitignore
.env
Save and close the file when you are finished editing.

Likewise, it’s a good precaution to add .env to a .dockerignore file, so that it doesn’t end up on your containers when you are using this directory as your build context.

Open the file:

nano .dockerignore
Add .env to the file:

~/wordpress/.dockerignore
.env
Below this, you can optionally add files and directories associated with your application’s development:

~/wordpress/.dockerignore
.env
.git
docker-compose.yml
.dockerignore
Save and close the file when you are finished.

With your sensitive information in place, you can now move on to defining your services in a docker-compose.yml file.

Step 3 — Defining Services with Docker Compose
Your docker-compose.yml file will contain the service definitions for your setup. A service in Compose is a running container, and service definitions specify information about how each container will run.

Using Compose, you can define different services in order to run multi-container applications, since Compose allows you to link these services together with shared networks and volumes. This will be helpful for our current setup since we will create different containers for our database, WordPress application, and web server. We will also create a container to run the Certbot client in order to obtain certificates for our webserver.

To begin, open the docker-compose.yml file:

vim docker-compose.yml
Add the following code to define your Compose file version and db database service, 
below your db service definition, add the definition for your wordpress application service, 
Next, below the wordpress application service definition, add the following definition for your webserver Nginx,
below your webserver definition, add your last service definition for the certbot service
Below the certbot service definition, add your network and volume definitions:

version: '3'

services:
  db:
    image: mysql:8.0
    container_name: db
    restart: unless-stopped
    env_file: .env
    environment:
      - MYSQL_DATABASE=wordpress
    volumes: 
      - dbdata:/var/lib/mysql
    command: '--default-authentication-plugin=mysql_native_password'
    networks:
      - app-network

  wordpress:
    depends_on: 
      - db
    image: wordpress:5.1.1-fpm-alpine
    container_name: wordpress
    restart: unless-stopped
    env_file: .env
    environment:
      - WORDPRESS_DB_HOST=db:3306
      - WORDPRESS_DB_USER=$MYSQL_USER
      - WORDPRESS_DB_PASSWORD=$MYSQL_PASSWORD
      - WORDPRESS_DB_NAME=wordpress
    volumes:
      - wordpress:/var/www/html
    networks:
      - app-network

  webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - wordpress:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - certbot-etc:/etc/letsencrypt
    networks:
      - app-network

  certbot:
    depends_on:
      - webserver
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot-etc:/etc/letsencrypt
      - wordpress:/var/www/html
    command: certonly --webroot --webroot-path=/var/www/html --email sammy@example.com --agree-tos --no-eff-email --staging -d example.com -d www.example.com

volumes:
  certbot-etc:
  wordpress:
  dbdata:

networks:
  app-network:
    driver: bridge
	

Step 4 — Obtaining SSL Certificates and Credentials
We can start our containers with the docker-compose up command, which will create and run our containers in the order we have specified. If our domain requests are successful, we will see the correct exit status in our output and the right certificates mounted in the /etc/letsencrypt/live folder on the webserver container.

Create the containers with docker-compose up and the -d flag, which will run the db, wordpress, and webserver containers in the background:

docker-compose up -d
You will see output confirming that your services have been created:

Output
Creating db ... done
Creating wordpress ... done
Creating webserver ... done
Creating certbot   ... done
Using docker-compose ps, check the status of your services:

docker-compose ps

