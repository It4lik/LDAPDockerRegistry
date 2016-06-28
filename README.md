# LDAPDockerRegistry

Want to set up a lab to get a secure Docker Registry ? With encryption ? And authentication ? Good place to be.

This repo contains a DockerCompose file that will set up such an environment for you. 

* Authentication is done thanks to a Slapd container.  
* Proxying is provided by Nginx.  
* And the official Registry image (v2) is used.

Nginx is going to be your endpoint, who can authenticate people against the LDAP Server, and redirect to your private registry if authentication was successful.

## Prerequisites

* A docker host with docker-engine > 1.8 and the docker-compose binary.
Go [here](https://docs.docker.com/compose/install/) to get Compose if you didn't already.

* A SSL key/cert pair. You can easily get a self-signed certificate (and key) with the following : 
```
$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout web.key -out web.crt
```

* Some love.

## Getting Started

1. Clone the repo and cd in it :  
	``` bash  
	$ git clone https://github.com/It4lik/LDAPDockerRegistry  
	$ cd LDAPDockerRegistry  
	```

1. Edit the environment file, which contains informations about the LDAP server. Make it suit to your needs.  
	``` bash  
	$ vi ./env/ldap.env  
	```

1. Move your .crt and .key files in certs directory :  
	``` bash  
	mv path-to-your.crt path-to-your.key ./nginx/certs  
	```

4. Edit Nginx configuration :
	``` bash
	$ vi ./nginx/config/nginx.conf
	```
	The following section contains informations about the LDAP connection :
	``` bash  
		ldap_server LDAP1
			url "ldap://<LDAP SERVER>/dc=your,dc=domain,dc=com?cn?sub?(objectClass=person)";
			binddn "cn=admin,dc=your,dc=domain,dc=com";
			binddn_passwd "<LDAP ADMIN PASSWORD>"; # This is the password you provided in the .env file
			bind_timeout 5s;
			request_timeout 5s;
			group_attribute member;
			group_attribute_is_dn on;
			require valid_user;
			satisfty all;
		}
	```

	&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;The next two sections also need a few changes (Server Name & .crt + .key path) :
	``` bash
		server {
			listen 		80;
			server_name	<SERVER NAME>;
			return		301 https://$server_name$request_uri;
		}

		server {
			listen          443;

			server_name	<SERVER NAME>;

			error_log	/var/log/nginx/error.log debug;
			access_log	/var/log/nginx/access.log;

			ssl on;
			ssl_certificate 	/etc/ssl/<YOUR .crt>;
			ssl_certificate_key 	/etc/ssl/<YOUR .key>;

			# disable any limits to avoid HTTP 413 for large image uploads
			client_max_body_size	0;
	
			# required to avoid HTTP 411: see Issue #1486 (https://github.com/docker/docker/issues/1486)
			chunked_transfer_encoding 	on;

			location / {
				auth_ldap "Please enter your ldap username";
				auth_ldap_servers LDAP1;

				root html;
				index index.html index.html;
			}

	```

5. Build your own Nginx container. Issue this from the LDAPDockerRegistry directory.  
	**The dot at the end of the line matters. Don't forget it.**  
	``` bash  
	$ cd nginx && docker build --tag=ldap_nginx .  
	# This may take a while, be patient, go grab some coffee.  
	```

6. Start the containers. This assumes you got docker-compose in your $PATH.  
Once more, execute this command from the LDAPDockerRegistry directory.  
	``` bash  
	$ docker-compose up  
	```

7. Add a test user to the LDAP server.  
The LDAP object your create **MUST be of the 'Person' type** if you haven't edit the LDAP connection string in the Nginx configuration.  

	To do this, multiple options :
	  * Get the **LDAP utilities** (go search for them with your favorite packet manager) and do a ldapadd
	  * Some **WebUI**. You can quickly setup one with the following (edit the variables to suit to your needs) :  
	``` bash
	$ docker run -p <HOST PORT>:80 -p <HOST PORT>:443 -e LDAP_HOST=<LOCAL LDAP IP ADDRESS> -e LDAP_BASE_DN=dc=your,dc=domain,dc=com -e LDAP_LOGIN_DN=cn=admin,dc=your,dc=domain,dc=com -d windfisch/phpldapadmin
	# And then, go check it with your browser using the ports you defined
	```  
	**NB :** The *\<LOCAL LDAP IP ADDRESS\>* is the private address of your LDAP container. You can [Docker inspect](https://docs.docker.com/engine/reference/commandline/inspect/) it to know it.


## Testing
Go ahead and log into your brand new and beautiful Docker registry with his magnificent Nginx + LDAP backend. 
``` bash
$ docker login https://<NGINX HOST>  # Do not provide any email address. And try fake credentials, just to be sure...
...
Login Succeeded
```

You can now interact with your own private registry. Congrats. 
``` bash
$ docker pull hello-world  # grab some lightweight image !
$ docker tag hello-world <YOUR HOST>/hw # tag it !
$ docker push <YOUR HOST>/hw # push it !
```
You should now be able to pull this image from another logged-in docker host. 

You can also setup UIs to explore your registry, like [this one](https://hub.docker.com/r/atcol/docker-registry-ui/) or [this one](https://hub.docker.com/r/hyper/docker-registry-web/). 

Happy pulls and pushes :)

---



# A word about self-signed certificate 
Docker doesn't like self-signed certificate. **Really**.  

* If you choose to use a self-signed certificate, you're probably got some trouble when trying to connect to your private registry.

	You need to add the *--insecure-registry* to your *DOCKER_OPTS*. More on that [here](https://docs.docker.com/registry/insecure/) for most of Linux distros. <return> 

	For some others (like CentOS 7), you can edit the Docker service definition (search for it with a *find* command or something). Edit the ExecStart line : 
	``` bash
	ExecStart=/usr/bin/docker daemon --insecure-registry <IP ADDRESS OF YOUR NGINX PROXY> -H fd://
	```
---
* If you get the following when trying to login : 
	```
	Error response from daemon: invalid registry endpoint https://<IP>/v0/: unable to ping registry endpoint https://<IP>/v0/
	v2 ping attempt failed with error: Get https://<IP>/v2/: x509: cannot validate certificate for IP because it doesn't contain any IP SANs
	 v1 ping attempt failed with error: Get https://<IP>/v1/_ping: x509: cannot validate certificate for IP because it doesn't contain any IP SANs. If this private registry supports only HTTP or HTTPS with an unknown CA certificate, please add `--insecure-registry 192.168.56.103` to the daemon's arguments. In the case of HTTPS, if you have access to the registry's CA certificate, no need for the flag; simply place the CA certificate at /etc/docker/certs.d/<IP>/ca.crt
	```

	You can solve this error doing one of the things that are recommended by the error output. Go grab the **.crt file** in your Docker Host and put in the /etc/docker/certs.d/IP/ repository. 
	That's can be done with the following : 
	```
	mkdir -p /etc/docker/certs.d/<IP>/ # replace with the IP you're trying to connect to
	scp IP:/~/LDAPDockerRegistry/nginx/certs/your.crt /etc/docker/certs.d/<IP>/ca.crt
	```
---