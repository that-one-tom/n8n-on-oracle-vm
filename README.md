# Setting up n8n using Ubuntu, Docker, NGINX + UFW on Oracle Cloud

## Sign up to Oracle Cloud
You can sign up for an Oracle Cloud account here: https://www.oracle.com/cloud/free/. You will need a credit card to verify your identity during sign up, but you will not be charged as long as you use the free tier.

## Create Compute Instance
Head to https://cloud.oracle.com/compute/instances and click on the `Create instance` button.

Fill out the form with the following information:
* Name: Any name you like
* Compartment: Default settings
* Placement: Any Availability Domain
* Image and shape: 
  * Image: Canonical Ubuntu 20.04 with the latest build date
  * Shape: Virtual Machine / Ampere / VM.Standard.A1.Flex
* Networking: Default settings
* Add SSH keys: Upload your public key files (if you need to generate one or are unsure about this, GitHub has a [great tutorial](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent))
* Boot volume: Default settings

Click Create once done.

## Connect to your instance
Once your instance has been provisioned, you should see a new entry on your [Instances page](https://cloud.oracle.com/compute/instances). Copy the public IP shown for your instance.

You can connect to it by opening a terminal and using the following command (replace the IP address with the one you copied):

```
ssh ubuntu@<public IP>
```

## Allow your instance to be accessed from the internet
On your [Instances page](https://cloud.oracle.com/compute/instances), click on the name of your new instance and then on the name of its `Virtual cloud network` (something like `vcn-20220106-1030` if you have not changed the default).

In the left sidebar, click on `Security Lists` and then the name of your Security List (with a name like `Default Security List for vcn-20220106-1030` if you have not changed the default).

Click on the `Add Ingress Rules` button and allow incoming traffic on port 80 (for HTTP) and 443 (for HTTPS) by setting the below values:
* Source CIDR: 0.0.0.0/0 (this means that any IP address can access the instance, if you'd like to limit access you would need to adjust [the value](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing#CIDR_notation))
* IP Protocol: TCP
* Destination Port Range: `80,443`

Confirm your settings by clicking the `Add Ingress Rules` button.

## Preparing your instance

### OS Updates
Before doing anything else, update your operating system by running these two commands:

```
sudo apt update
sudo apt upgrade
```

### Docker & Docker Compose
Next [install Docker](https://docs.docker.com/engine/install/ubuntu/), the container engine which will later run n8n, as well as [Docker Compose](https://docs.docker.com/compose/install/):

```
# Installing Prerequisites
sudo apt install ca-certificates curl gnupg lsb-release

# Add Docker's GPG key and repository
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package sources and install Docker
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io

# Allow the current user to run Docker commands without sudo
sudo usermod -aG docker ${USER}
sudo su - ${USER}

# Test Docker was installed correctly
docker run hello-world

# Install Docker Compose
sudo apt install docker-compose
```

### NGINX
The last piece to install is NGINX. This will be our reverse proxy accepting requests from the internet and forwarding them to n8n. It can easily be extended in case you want to run additional applications on the same instance.

```
sudo apt install nginx
```

Make sure NGINX is running:

```
# The below command should return `Active: active (running)` among other information.
sudo systemctl status nginx
```

### UFW
In this example setup, we will be using [UFW](https://help.ubuntu.com/community/UFW) to configure [iptables](https://help.ubuntu.com/community/IptablesHowTo). Oracle's Ubuntu image does, however, come with iptables-persistent which would need to be removed. To do so, simply run:

```
sudo apt remove iptables-persistent
```

At this stage, reboot your instance to make sure all previous changes have taken effect.

```
sudo reboot now
```

We are now ready to configure UFW. First make sure it knows about all your applications by running the below command. It should return `Nginx Full` and `OpenSSH` among the available applications.

```
sudo ufw app list
```

Now allow both `Nginx Full` and `OpenSSH` to be accessed from the internet:

```
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
```

Adjust `UFW`'s configuration to avoid accidentially opening ports when using Docker by running this command:

```
sudo nano /etc/ufw/after.rules
```

Now scroll down to the end of the file and add [this configuration snippet](https://github.com/chaifeng/ufw-docker#solving-ufw-and-docker-issues) to the end of the file:

```
# BEGIN UFW AND DOCKER
*filter
:ufw-user-forward - [0:0]
:ufw-docker-logging-deny - [0:0]
:DOCKER-USER - [0:0]
-A DOCKER-USER -j ufw-user-forward

-A DOCKER-USER -j RETURN -s 10.0.0.0/8
-A DOCKER-USER -j RETURN -s 172.16.0.0/12
-A DOCKER-USER -j RETURN -s 192.168.0.0/16

-A DOCKER-USER -p udp -m udp --sport 53 --dport 1024:65535 -j RETURN

-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 192.168.0.0/16
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 10.0.0.0/8
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 172.16.0.0/12
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 192.168.0.0/16
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 10.0.0.0/8
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 172.16.0.0/12

-A DOCKER-USER -j RETURN

-A ufw-docker-logging-deny -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW DOCKER BLOCK] "
-A ufw-docker-logging-deny -j DROP

COMMIT
# END UFW AND DOCKER
```

Exit the editor by pressing `Ctrl+X` and then confirm with `Y` when asked whether you want to save the changes made.

Lastly, enable `UFW`:

```
sudo ufw enable
```

## Test the webserver
On your local machine, open a browser and navigate to `http://<public IP>` (replace the IP address with the one you copied earlier). You should see the default Nginx landing page "Welcome to NGINX! (...)"

## Installing n8n and configuring NGINX

### Set Up DNS Record
This first step will look slighlty different different depending on the provider you choose so you might need to consult the respective documentation. Also it's worth keeping in mind that DNS records have a time to live (TTL) during which they will not be queried again. That means any change will not take effect immediately but could take several hours. 

To secure traffic to our n8n instance with [SSL/TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security) using free certificates from [Let's Encrypt](https://letsencrypt.org/), we need a domain name from a provider that let's us configure DNS settings. Namecheap, GoDaddy, or Google Domains are a few big names providing affordable domains, but there are many more and pretty much any of them will do.

Log in to the admin panel of your provider (where you manage your domain names) and create a DNS A record for your chosen (sub-)domain with a value of the public IP address of your Oracle Cloud VM instance.

### Install n8n
We will be using [Docker Compose](https://docs.docker.com/compose/overview/) to run n8n. The below commands will create a folder in which we then create the `docker-compose.yml` configuration file describing the application.

```
cd ~
mkdir n8n
cd n8n
nano docker-compose.yml
```

Inside the `nano` editor, paste the below example configuration and replace
* `<domain name>` with the domain you have set up including HTTPS, e.g. `https://n8n.mutedjam.com/`
* `<n8n user>` with the username you would like to use to access n8n
* `<n8n password>` with a password of your choice
* `Europe/Berlin` with the correct timezone (if you are not based in the Berlin timezone)

```
version: '2'

services:
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    container_name: n8n
    ports:
      - 5678:5678
    environment:
      - WEBHOOK_URL=<domain name>
      - GENERIC_TIMEZONE=Europe/Berlin
      - N8N_DIAGNOSTICS_ENABLED=false
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=<n8n user>
      - N8N_BASIC_AUTH_PASSWORD=<n8n password>
    volumes:
      - /home/ubuntu/n8n/n8n_data:/home/node/.n8n
```

Any other environment variables you would like to use can also be added under the `environment` section above. A full list is available in the [n8n documentation](https://docs.n8n.io/reference/environment-variables.html).

Exit the editor by pressing `Ctrl+X` and then confirm with `Y` when asked whether you want to save the changes made.

Now we can start n8n using the below command:

```
docker-compose up --detach
```

n8n should now start in the background. Give it a few moments to do so, then check that it has started correctly by looking at its output using below command:

```
docker logs n8n
```

The last lines should say `Editor is now accessible via: http://localhost:5678/` indicating that n8n is ready.

At this stage, you can already connect to n8n directly from your VM (e.g. by running `curl http://localhost:5678/`), but it is not yet exposed to the world.

### Configure NGINX

The last step is making your n8n instance available to the outside world. To do so, we create what NGINX calls a server block (a configuration block that defines how a server responds to requests):

```
cd /etc/nginx/sites-available/
sudo nano n8n.conf
```

Now insert a copy of the below example configuration and replace
* `<domain name>` with the domain you have set up (**but without HTTP/HTTPS**), e.g. `n8n.mutedjam.com`

```
server {
    server_name <domain name>;
    listen 80;

    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Connection '';
        proxy_http_version 1.1;
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;
    }
}
```

Exit the editor by pressing `Ctrl+X` and then confirm with `Y` when asked whether you want to save the changes made.

Now link the file we have just created to the `sites-enabled` folder and afterwards test your configuration to make sure everything is understood by NGINX:

```
sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
sudo nginx -t
```

You should see an output ending with `nginx: configuration file /etc/nginx/nginx.conf test is successful`. We can now proceed with reload NGINX to apply the newly added configuration:

```
sudo systemctl reload nginx
```

At this stage, you should be able to access your n8n instance from outside by opening `http://<domain name>` in a browser on your local machine (replace the domain name with the one you copied earlier). You probably don't want to enter your username/password yet when prompted, as at the moment the connection between your browser and your Oracle VM is a plain text connection without any encryption.

### Secure your n8n instance with SSL/TLS

In this final step, we will issue a certificate from Let's Encrypt to secure connections to our n8n instance. First, install `certbot` which takes care of the painful stuff for us:

```
sudo apt install certbot python3-certbot-nginx
```

Then, request a certificate for your domain (one last time replace `<domain name>` with the domain you have set up, e.g. `n8n.mutedjam.com`):

```
sudo certbot --nginx -d <domain name>
```

certbot will ask you a few questions. You can answer with the default values, or you can change them to your liking. It will then obtain the certificate for you and finally asking `Please choose whether or not to redirect HTTP traffic to HTTPS (...)`. You might want to hit `2` here (Redirect) to make sure all traffic to your instance is secured going forward.

You should now be able to access n8n via a secure connection by navigating to `https://<domain name>` in your browser.
