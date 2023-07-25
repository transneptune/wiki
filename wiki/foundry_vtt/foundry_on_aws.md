# Setting up Foundry on AWS

## Prerequisites

 - AWS account
 - Licensed copy of Foundry VTT
 - A domain name you control, through a registrar of your choice
 - POSIX-compliant local OS (macOS or Linux, let's be real)
    - If you're using Windows, the steps around how you connect to the EC2
      instance will differ. I do not use Windows and do not know how they will
      differ.
 - Basic familiarity with using a shell, and ssh

## Costs

 - AWS: ~$0 (all free-tier eligible)
 - Foundry VTT: $50 one-time license fee
 - Domain name: ~$15/year

## Overall plan of action

 - AWS:
     - Set up EC2 instance (t2.micro)
     - Set up ElasticIP
     - Set up SecurityGroup rules (including opening port 30000!)
     - Tag these all for easy finding later
 - On local machine:
     - Set up SSH key and config
 - On domain registrar:
     - Set up DNS
 - On EC2 instance:
     - Set up nginx (just to handle some simple HTTP redirects)
     - Set up HTTPS cert (via certbot)
     - Set up HTTP -> HTTPS forwarder (automatic with nginx + certbot)
     - Set up HTTPS -> 30000 forwarder
     - Set up Foundry to use certbot certs
     - Set up a system user for Foundry to run as
     - Set up systemd to autorun Foundry

## On AWS

In the AWS browser console, set up an EC2 instance with the following properties:

 - Name: Foundry
 - Tags:
    - Project: Foundry
 - AMI/OS: Ubuntu 22.04 LTS
 - Instance type: t2.micro
 - Key pair: create new key pair
    - Name: foundry
    - Key pair type: RSA
    - Private key format: .pem
 - Network settings:
    - Create security group
    - Allow SSH traffic from Anywhere
    - Allow HTTPS traffic from the internet
    - Allow HTTP traffic from the internet
 - Configure storage: defaults OK
 - Advanced details: default OK

The keypair `.pem` file will be downloaded to your local computer. **DO NOT
LOSE THIS!** It is your only way to access the EC2 instance. We will come back
to this in "On local machine", below.

Now that the EC2 instance is up and running, we need to give it a stable IP
address. In the EC2 area, under "Network & Security" on the sidebar, choose
"Elastic IPs". Allocate a new one, assigning it to your newly made EC2
instance. Note down the IP address; you will use it under "On your DNS
registrar", below. Tag this Elastic IP with "Project: Foundry" also.

Finally, also under "Network & Security" in the sidebar, go to "Security
Groups". You will see a number of groups, but one should have a "Security group
name" like `launch-wizard-1`. This should be the one associated with your EC2
instance. Follow the link to the detail view for this one, and add one rule to
the "Inbound rules" section:

 - Port range: `30000`
 - Source: `0.0.0.0/0`

Save that, and name it "Foundry". Tag the security group with "Project:
Foundry" too. This will allow the server to receive traffic on port 30000, the
default port Foundry VTT uses.

## On your local machine

Open a shell. Move the `foundry.pem` key into your `~/.ssh` directory, or wherever you choose to store your SSH keys. Edit `~/.ssh/config` to include a stanza like this:

```
Host foundry
    HostName <the elastic IP from above, or your DNS name once you've set that up>
    User ubuntu
    IdentityFile ~/.ssh/foundryvtt.pem
```

Now you can connect to your EC2 instance over SSH by running:

```
ssh foundry
```

Do so, to confirm that everything's working.

## On your domain registrar

Using your domain registrar of choice, set up a domain or subdomain pointing to
the Elastic IP. This should be an `A` record. We will use `play.example.com` as
the example domain for the rest of this guide.

## On the EC2 instance

Now we're ready to configure the EC2 instance.

First, ssh in to it, or use the ssh session you created above if that's still running.

Now, we'll need to do a number of things:

1. Set up nginx to handle basic HTTP and HTTPS redirects.
2. Set up cerbot to issue and renew certificates for HTTPS.
3. Create a system user for Foundry to run as.
4. Install Foundry VTT
5. Set up Systemd to automatically run Foundry.

### Setting up nginx

Install nginx:

```
sudo apt update
sudo apt install nginx
```

Test that it is listening and reachable by going to `http://play.example.com`
in a browser.

Consider following this guide on "[Setting up server
blocks](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-22-04#step-5-setting-up-server-blocks-recommended)".

### Setting up certbot

Install certbot:

```
sudo snap install core
sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

Then run it to get a cert. This should automatically interact with nginx and
configure it to use the acquired certs.

```
sudo certbot --nginx -d play.example.com
```

Consider following this guide on "[Verifying Cerbot
auto-renewal](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-22-04#step-5-verifying-certbot-auto-renewal)".

Verify that this has worked by visiting `https://play.example.com` in a
browser, and checking that you get no warnings or errors about an invalid
certificate.

Certbot will produce certs in these locations:

```
/etc/letsencrypt/live/play.example.com/fullchain.pem
/etc/letsencrypt/live/play.example.com/privkey.pem
```

which in turn are links to certs like these:

```
/etc/letsencrypt/archive/play.example.com/fullchain1.pem
/etc/letsencrypt/archive/play.example.com/privkey1.pem
```

You will need to change permissions on some of the directories and files
produced, in order for Foundry VTT to be able to read them eventually:

```
sudo chmod g+r,g+x /etc/letsencrypt/live/
sudo chmod g+r,g+x /etc/letsencrypt/archive/
sudo chmod g+r /etc/letsencrypt/archive/play.example.com/privkey1.pem  # Note to self: this may need to change as certs get rolled. Possible future pain point.
```

### Setting up a system user for Foundry

Make a system user, and add it to the right groups:
```
adduser --system --no-create-home foundry  # Note to self: can this be done better?
sudo usermod -aG ubuntu foundry  # To see Foundry files
sudo usermod -aG root foundry    # To see Certbot certs
```

We'll be setting up Foundry's data and core directories as the default `ubuntu`
user, for ease of working with them day-to-day, so we need to add the `foundry`
user to the `ubuntu` group. We also need the `foundry` user to be able to see
the cert files produced above, so we add it to the `root` group.

### Install Foundry

Install Foundry's runtime dependencies:

```
curl -sL https://deb.nodesource.com/setup_18.x | sudo bash -
sudo apt install -y libssl-dev unzip
sudo apt install -y nodejs
```

Then download and install Foundry itself:

```
cd $HOME
mkdir foundryvtt
mkdir foundrydata

# Install the software
cd foundryvtt
wget -O foundryvtt.zip "<foundry-website-download-url>"
unzip foundryvtt.zip
```

See notes on where to get the download URL and the license key
[here](https://foundryvtt.com/article/installation/).

Run it to test that all permissions and configuration are correct:

```
sudo -u foundry node /home/ubuntu/foundryvtt/resources/app/main.js --dataPath=/home/ubuntu/foundrydata
```

If there are no errors in the terminal, visit `https://play.example.com:30000`
to verify that it's serving content. Do not visit the `https` version yet; that
hasn't been configured. If that's all good, you'll want to configure a few
things before you stop Foundry on the server.

First, you'll have to insert your license key.

Second, you'll want to set a few values in Configuration.

 - Administrator password: set this to something secure that you will remember.
 - SSL Certificate: `/etc/letsencrypt/live/play.example.com/fullchain.pem`
 - SSL Private Key: `/etc/letsencrypt/live/play.example.com/privkey.pem`

Once that's all done, hit ctrl-C in the terminal to kill the running Foundry
process, if the configuration steps haven't automatically killed the process
anyway.

#### Tweak Nginx forwarding

You'll need to set up one more thing to enable forwarding from
`https://play.example.com` to `https://play.example.com:30000`. Add the
following to your nginx configuration:

```
if ($host = play.example.com) {
    # Only a 302 in case we wanna easily change this later:
    return 302 https://$host:30000$request_uri;
} # Foundry!
```

near the end of your server block listening on `443`. You can't do this
usefully until you've set up your certificates in the Foundry configuration,
which is why we deferred it until this moment!

### Running Foundry automatically

Make a Systemd unit file to manage autorestarting Foundry in the background:

```
sudo cat << EOF > /etc/systemd/system/foundry.service
[Unit]
Description=Foundry
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=foundry
ExecStart=node /home/ubuntu/foundryvtt/resources/app/main.js --dataPath=/home/ubuntu/foundrydata

[Install]
WantedBy=multi-user.target
EOF
```

Enable and start the Foundry Systemd service:

```
sudo systemctl daemon-reload
sudo systemctl enable foundry
sudo systemctl start foundry
```

Now check that it's all working by going to `http://play.example.com` and
seeing that you get forwarded to `https://play.example.com:30000`, and are
served the correct content. Congrats!
