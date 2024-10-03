
# Task 2: Site deployment

DevOps Engineer Assessment 2 from Caplinq



## Configuring Server

This deployment uses Linode as the cloud provider. The VM details are as follows

Region: SG, Singapore (ap-south)
OS: Ubuntu 22.04 LTS
Linode Plan: Dedicated 4 GB  
Other settings: default/untouched


## Get started

### Accessing the server

Run `ssh-keygen` and enter the details while configuring the vm, for this particular vm the following detail is used to enter the vm

```
ssh root@139.162.7.197
```

Then its best to create user first and give sudo permission use the following commands
```
sudo apt update
adduser <name of user>
usermod -aG sudo <name of user>         # Grant sudo permission

```

After creating user configure the firewall first to connect using ssh safely, and then switch to user

### Firewall (UFW)
```
ufw allow OpenSSH  
ufw enable  
ufw status      # should show OpenSSH & OpenSSH (v6)
su <name of user>                       # Switch to user instead of using root
cd                                      # to get out of root folder
```

### Swap Space
Increase swap space for more stable, preferably less than equal or max of 4gb RAM

```
sudo fallocate -l 4G /swapfile  
sudo chmod 600 /swapfile  
sudo mkswap /swapfile  
sudo swapon /swapfile  
cat /proc/swaps     # to check if partition is created
```

Install web server -> NGINX

### Nginx
```
sudo apt update  
sudo apt install nginx
sudo ufw allow 'Nginx Full'  
```

Install certbot for ssl/https connection
### Certbot
```
sudo snap install --classic certbot  
sudo ln -s /snap/bin/certbot /usr/bin/certbot  
sudo /etc/init.d/nginx stop                     # important to stop before issuing cert
sudo certbot --nginx -d demo.clinicprime.online # this domain was used on this particular deploymnet
sudo chmod -R 755 /etc/letsencrypt

```

## Angular

For this task i used Angular for as the app for deployment. For native angular installation the following commands are used

### Nodejs
```
sudo apt install nodejs  
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash  
source ~/.bashrc  
nvm install v22.9.0  
sudo apt install npm            # install npm to install angular
```

### Angular
```
sudo npm install -g @angular/cli  
ng --version                    # should be 18.2.7
ng new <project_name>             # create new project
```

Before deploying the project it should be built first using
```
cd <project_name>  
npm run build --production
```

The build should return the `Output Location` which will be used on the next commands

Before continuing in setting up NGINX it is important to give permission to the project

### Give right permission to nginx
```
sudo chown -R www-data:www-data /home/caplinq/<Output Location>/browser  
sudo chmod -R 755 /home/caplinq/<Output Location>/browser  
sudo find /home/caplinq/<Output Location>/browser -type d -exec chmod 755 {} \;  
sudo find /home/caplinq/<Output Location>/browser -type f -exec chmod 644 {} \;  
sudo chmod 755 /home/caplinq  
sudo chmod 755 /home/caplinq/
sudo chmod 755 /home/caplinq/<project_name> 
sudo chmod 755 /home/caplinq/<project_name>/dist  
sudo chmod 755 /home/caplinq/<project_name>/dist/demo-site  
```




## Nginx Configuration Breakdown

Once the project is ready for deployment the  next step is configuring Nginx, the config is located inside `/etc/nginx/conf.d/nginx.conf`

If `nginx.conf` does not exist, create the file using `touch` for directly use `nano nginx.conf` then paste the following:

```
limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;

# server blocks

server {

        listen 443 ssl;

        server_name
                demo.clinicprime.online
                ;

        root /home/caplinq/production/demo-site/dist/demo-site/browser;

        # SSL certificate
        ssl_certificate /etc/letsencrypt/live/demo.clinicprime.online/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/demo.clinicprime.online/privkey.pem;
        ssl_session_timeout  5m;
        ssl_session_cache shared:SSL:10m;
        ssl_session_tickets off;
        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:HIGH:!aNULL:!MD5';
        ssl_ecdh_curve secp384r1;
        ssl_prefer_server_ciphers on;

        # Server header
        add_header X-Frame-Options "SAMEORIGIN";
        add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";

        # Optimization/DDoS  mitigation
        sendfile on;
        keepalive_timeout 15;
        client_max_body_size 50m;
        client_body_buffer_size 16K;
        client_header_buffer_size 1k;

        location / {
                limit_req zone=one burst=20 nodelay;    #Rate limit
#               try_files $uri $uri/ /index.html;
        }

        location /api/ {
                proxy_pass http://localhost:3000/;  # Your backend API server
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;

                # WebSocket support (optional)
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
        }

        # Logging
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;


}

# http to https redirect
        server {
                listen 80;
                server_name
                        demo.clinicprime.online
                        ;

                return 301 https://$host$request_uri;
        }
```

Next go to `/etc/nginx/site-enabled`, edit the `default` and delete all the server block after the first one



To verify the config and run nginx (make sure to run in sudo)
```
sudo nginx -t                        # Verify if config has no errors
sudo systemctl restart nginx    # if nginx -t does not result in error
```