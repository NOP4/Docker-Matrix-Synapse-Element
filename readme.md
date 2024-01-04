# docker-compose for Matrix+Synapse+Element+SynapseAdmin using Traefik Reverse Proxy autoconfiguration, with let's encrypt, HSTS, and fail2ban.

This stack will deploy everything necessary to get a working Matrix installation.

Your installation will have:
- Synapse server
- Element web access
- Synapse-admin
- Redis
- Fail2ban
- Emails
- Federation
- Automatic update
- Password complexity requirements

The information provided here uses example.com as an example. You need to have your own domain name and a docker server, preferably with Portainer installed.


## Step 1 - Create DNS entries

In your DNS server, create entries for:
- "A" entry for: `matrix.example.com`
- "A" entry for: `element.example.com`
- "SRV" entry for: `_matrix._tcp.matrix` &emsp; `0 443 matrix.example.com`

You'll need to ajust with your own domain name and wanted subdomain.


## Step 2 - Deploy the Traefik stack and add the matrix federation entrypoint.

See https://github.com/NOP4/Docker-Traefik-Reverse-Proxy for Traefik deployment.

Then add the following line in the "command" section of your Traefik docker-compose stack:

`- "--entrypoints.matrixfederation.address=:8448"`

Restart the Traefik stack.


## Step 3 - Firewall configuration

Port 8448 is used for Feredaction, we need to allow it in our firewall rules:

`ufw allow 8443`


## Step 4 - Create linux users and groups for containers


In this example, we will create users and groups starting at 900. Adapt to your liking the names, UID and GID.

`addgroup docker_example_matrix --gid 900`

`adduser docker_example_matrix --uid 900 --gid 900 --shell /dev/null --no-create-home --disabled-login`

`addgroup docker_example_element --gid 901`

`adduser docker_example_element --uid 901 --gid 901 --shell /dev/null --no-create-home --disabled-login`

`addgroup docker_example_element_admin --gid 902`

`adduser docker_example_element_admin --uid 902 --gid 902 --shell /dev/null --no-create-home --disabled-login`

## Step 5 - Initial deployment

Use the provided docker-compose.yml file.

This file assumes you deploy:
- Synapse server at: matrix.example.com
- Element server at: element.matrix.com
- Synapse Admin at: matrix.example.com/synapseadmin 

You will need to change a few items:
- `POSTGRES_PASSWORD=STRONG-PASSWORD` set your own strong password
- `SYNAPSE_SERVER_NAME=matrix.example.com` change to your own domain name
- change all `matrix.example.com` to your domain name
- change all `element.exaple.com` to your domain name
- change all `example.com` to match your domain name
- change all `*.example.com` to match your domain name
- adapt traefik labels changing `example` to your domain name (optional)
- change `MY-ADMIN-IP-ADDRESS/32` to the IP address from which you can access the admin interface
- adapt user and groups according to what you created on previous step

Deploy the stack then stop it when the config files and database entries are created. 


## Step 6 - Update deployment

Edit the docker-compose.yml and comment or remove line `command: "generate"`

Modify the synapse `config.json` and element `homeserver.yaml` files to look similar to the two provided examples.

In Synapse `config.json` you'll have to:

- check your server name and URL.

In Element `homeserver.yaml` you'll have to:

- check server name and URL
- change database to psycopg2 instead of sqlite
- set registration random shared secret
- set email SMTP server settings
- set password pepper random string
- ajust password policy as required

In examples provided, your Element web page will be force to only serve your domain (that's preferable), and the public registration is disabled. This can be easily changed in the config files.


Redeploy stack. You can delete the sqlite database file.


## Step 7 - Check with federation tester

Check with Matrix federation tester that the federation is working fine: https://federationtester.matrix.org


## Step 8 - Create admin user

In your portainer GUI, start the command line for your synapse server and execute this command to create an admin user. Make it admin when prompted. All other users will be created using the synapseadmin URL.
`register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008`


## Set 9 - Create end users

Connect to your server admin URL: https://matrix.example.com/synapseadmin

Login with your admin user and create your end user accounts. Don't forget to add their email address.

You can create the accounts with random passwords, as the users will be able to use the "forgot password" option to set their own.

They should receive a welcome email.


## Setp 10 - Automatic update

Deploy a watchtower stack for automatic update.
It as as simple as deploying this stack:

```
version: '3'

services:
  watchtower:
    image: containrrr/watchtower
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /etc/timezone:/etc/timezone:ro
    environment:
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_LABEL_ENABLE=true
      - WATCHTOWER_INCLUDE_RESTARTING=true
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
```


## Step 11 - Install and setup fail2ban

If not alredy installed on your server, install fail2ban.

`apt install fail2ban`

Copy the provided files in your /etc/fail2ban/ folders, you'll need to edit them to adapt path to your matrix path logfiles.

Activate and start fail2ban:

`systemctl enable fail2ban`

`systemctl start fail2ban`

Get the status of your filtering:

`fail2ban-client status example-matrix-synapse`

Test some bad passwords to check the filtering is working fine. Then unban the IP with:

`fail2ban-client set example-matrix-synapse unbanip IP-TO-UNBAN`

