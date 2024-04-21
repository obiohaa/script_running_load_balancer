## Automating Load balancer configuration with Shell scripting

In our initial project we deployed two backend servers, with a load balancer distributing traffic
across the web servers. This was done from the terminal when we connect to them. Here we will do it
via automation. We will do this by writing a shell script that when ran, all that we did manually
will be done automatically.

### Deploying and Configuring the Web servers

1. We provision an EC2 instance running ubuntu 20.04.
2. We open port 8000 to allow traffic from anywhere using the security group
3. we connect to the web server via the terminal using SSH client
4. Open a file, paste the below script in it and close using the code

```
sudo vi install.sh
```

- close the file by using ` esc + :wqa!`

5. W then change the permission on the file to make it executable.

```
sudo chmod +x install.sh
```

6. Lastly we run the script

```
./install.sh PUBLIC_IP
```

```
#!/bin/bash

####################################################################################################################
##### This automates the installation and configuring of apache webserver to listen on port 8000
##### Usage: Call the script and pass in the Public_IP of your EC2 instance as the first argument as shown below:
######## ./install_configure_apache.sh 127.0.0.1
####################################################################################################################

set -x # debug mode
set -e # exit the script if there is an error
set -o pipefail # exit the script when there is a pipe failure

PUBLIC_IP=$1

[ -z "${PUBLIC_IP}" ] && echo "Please pass the public IP of your EC2 instance as an argument to the script" && exit 1

sudo apt update -y &&  sudo apt install apache2 -y

sudo systemctl status apache2

if [[ $? -eq 0 ]]; then
    sudo chmod 777 /etc/apache2/ports.conf
    echo "Listen 8000" >> /etc/apache2/ports.conf
    sudo chmod 777 -R /etc/apache2/

    sudo sed -i 's/<VirtualHost \*:80>/<VirtualHost *:8000>/' /etc/apache2/sites-available/000-default.conf

fi
sudo chmod 777 -R /var/www/
echo "<!DOCTYPE html>
        <html>
        <head>
            <title>My EC2 Instance</title>
        </head>
        <body>
            <h1>Welcome to my EC2 instance</h1>
            <p>Public IP: "${PUBLIC_IP}"</p>
        </body>
        </html>" > /var/www/html/index.html

sudo systemctl restart apache2
```

![server1](./images/server1.PNG) and ![server1](./images/server2.PNG)

### Deploying Nginx as a load balancer using shell script

We will provision another EC2 instance running on port 80 to anywhere using the security group, then
we connect to the load balancer via the terminal.

1. On our terminal we open a file nginx.sh `sudo vi nginx.sh`
2. We copy and paste the below code into the file created
3. we close the file using the command `esc + :wqa!`
4. We change the file permission to make the file executable ` sudo chmod +x nginx.sh`
5. Then we run the script `./nginx.sh IP server1_url server2_url`

```

#!/bin/bash

######################################################################################################################
##### This automates the configuration of Nginx to act as a load balancer
##### Usage: The script is called with 3 command line arguments. The public IP of the EC2 instance where Nginx is installed
##### the webserver urls for which the load balancer distributes traffic. An example of how to call the script is shown below:
##### ./configure_nginx_loadbalancer.sh PUBLIC_IP Webserver-1 Webserver-2
#####  ./configure_nginx_loadbalancer.sh 127.0.0.1 192.2.4.6:8000  192.32.5.8:8000
#############################################################################################################

PUBLIC_IP=$1
firstWebserver=$2
secondWebserver=$3

[ -z "${PUBLIC_IP}" ] && echo "Please pass the Public IP of your EC2 instance as the argument to the script" && exit 1

[ -z "${firstWebserver}" ] && echo "Please pass the Public IP together with its port number in this format: 127.0.0.1:8000 as the second argument to the script" && exit 1

[ -z "${secondWebserver}" ] && echo "Please pass the Public IP together with its port number in this format: 127.0.0.1:8000 as the third argument to the script" && exit 1

set -x # debug mode
set -e # exit the script if there is an error
set -o pipefail # exit the script when there is a pipe failure


sudo apt update -y && sudo apt install nginx -y
sudo systemctl status nginx

if [[ $? -eq 0 ]]; then
    sudo touch /etc/nginx/conf.d/loadbalancer.conf

    sudo chmod 777 /etc/nginx/conf.d/loadbalancer.conf
    sudo chmod 777 -R /etc/nginx/


    echo " upstream backend_servers {

            # your are to replace the public IP and Port to that of your webservers
            server  "${firstWebserver}"; # public IP and port for webserser 1
            server "${secondWebserver}"; # public IP and port for webserver 2

            }

           server {
            listen 80;
            server_name "${PUBLIC_IP}";

            location / {
                proxy_pass http://backend_servers;
            }
    } " > /etc/nginx/conf.d/loadbalancer.conf
fi

sudo nginx -t

sudo systemctl restart nginx
```

- Server A ![saverA](./images/saverA.PNG)

- Server B ![saverA](./images/saverA.PNG)

- Load Balancer A

![load balancer](./images/loadbalancer.PNG)

- Load Balancer B ![load balancer](./images/load_balancer_B.PNG)

# END
