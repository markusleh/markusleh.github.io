---
layout: post
title:  Configuring Keycloak with Apache2
category: howto
description: How to configure Keycloak behind Apache2
excerpt: This post consists of configuring Keycloak behind Apache reverse proxy + Let's encrypt
---

This post consists of the following parts:
1. Downloading and configuring Keycloak (IdP)
2. Downloading and configuring Apache2 to act as a reverse proxy in front of Keycloak (IdP)
3. Bonus: Configuring Let's encrypt (IdP)

IdP: identity provider
idp.example.com in this post

# Keycloak (IdP)
1. Download and configure Keycloak (Ubuntu 16.04)

1.1 Install Java
```
sudo apt install openjdk-8-jdk
```

1.2 Install Keycloak
```
wget https://downloads.jboss.org/keycloak/3.2.1.Final/keycloak-3.2.1.Final.zip
unzip keycloak-3.2.1.Final.zip
mv keycloak-3.2.1.Final.zip /opt/keycloak/
```
Keycloak is now "installed" in /opt/keycloak/. Keycloak runs basically just on java scripts and no binaries are needed.

1.3 Add user for Keycloak
```
/opt/keycloak/bin/add-user-keycloak.sh -u username
```

1.4 Start the server
```
/opt/keycloak/bin/bin/standalone.sh -b bind-address
```
Changing the bind-address to an IP address that is reachable from your configuration machine. Note that if you specify non-local address you cannot log 
in to the server without https. If you follow this tutorial further, set the bind address to `127.0.0.1`

In the next step we configure HTTPS with Apache2 as a reverse proxy in front of Keycloak. If you don't want to do that, please see https://keycloak.gitbooks.io/documentation/server_installation/topics/network/https.html
 for how to configure HTTPS in Keycloak.
 
1.5 Configure Keycloak for Apache (or any reverse proxy)
 Set the following parameters in `/opt/keycloak/standalone/configuration/standalone.xml`
```
<subsystem xmlns="urn:jboss:domain:undertow:3.0">
  ...
  <server name="default-server">
    <!------- Change this line ----->
    <http-listener name="default" socket-binding="http" proxy-address-forwarding="true" redirect-socket="proxy-https" />
  ...
<socket-binding-group name="standard-sockets" default-interface=" ...
  ...
  <!------- Add this line ----->
  <socket-binding name="proxy-https" port="443"/>
  ...
```
1.6 Set up Keycloak as a service
Download the service scripts from:
https://gist.github.com/markusleh/eb2f1c112b71608c56b1a801f0f70f78

Make sure to save both
- keycloak-initd
- keycloak-defaults

Place them in the following locations:
```
mv keycloak-defaults  /etc/default/keycloak
mv keycloak-initd      /etc/init.d/keycloak
```

Set the permissions and enable the new service
```
sudo chown root:root /etc/init.d/wildfly
sudo chmod +X /etc/init.d/keycloak
# Enable the new service
update-rc.d keycloak defaults
update-rc.d keycloak enable
# Create logging directory 
sudo mkdir -p /var/log/keycloak
# Create user to run keycloak as 
sudo useradd --system --shell /bin/false keycloak
# Set permissions for the new user
sudo chown -R keycloak /opt/keycloak/
sudo chown -R keycloak /var/log/keycloak/
# Finally, start the service
sudo service keycloak start
```
Keycloak should now be running

2.0 Configure apache (IdP)

2.1 Install apache2
```
sudo apt install apache2
```
2.2 Configure certificate for apache
Visit https://certbot.eff.org/
For instructions how to set up Let's encrypt. They have a great tool and a guide so I won't post it here. They make sure it's relevant 
and updated for all environments.

2.3. Configure Apache to act as a reverse proxy

Put the following in 
/etc/apache2/sites-available/000-default-le-ssl.conf

Under <VirtualHost *:443>
```
    ProxyPreserveHost On
    SSLProxyEngine On
    SSLProxyCheckPeerCN on
    SSLProxyCheckPeerExpire on
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Port "443"
    ProxyPass / http://127.0.0.1:8080/
    ProxyPassReverse / http://127.0.0.1:8080/
    
```
