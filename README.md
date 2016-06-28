# LDAPDockerRegistry

Want to set up a lab to get a secure Docker Registry ? With encryption ? And authentication ? Good place to be.

This repo contains a DockerCompose file that will set up such an environment for you. 

Authentication is done thanks to a Slapd container. <return>
Proxy is provided by Nginx. <return>
And the official Registry image (v2) is used.

## Prerequisites

- A docker host with docker-engine > 1.8 and the docker-compose binary.
Go [here](https://docs.docker.com/compose/install/) to get Compose if you didn't already.

- A SSL key/cert pair. You can easily get a self-signed certificate (and key) with the following : 
```
$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout web.key -out web.crt
```

- Some love.

## Getting Started

* Clone the repo and cd in it
``` bash
$ git clone https://github.com/It4lik/LDAPDockerRegistry
$ cd LDAPDockerRegistry
```

* Edit the environment file, which contains informations about the LDAP server. Make it suit to your needs.
``` bash
$ vi ./env/ldap.env
```

* Edit Nginx configuration
 * This section contains informations about the LDAP connection
``` bash
	ldap_server LDAP1 {
		url "ldap://<LDAP SERVER>/DC=your,DC=domain,DC=com?cn?sub?(objectClass=person)";
		binddn "<LDAP USER NAME";
		binddn_passwd "<LDAP USER PASSWORD>";
		bind_timeout 5s;
		request_timeout 5s;
		group_attribute member;
		group_attribute_is_dn on;
		require valid_user;
		satisfty all;
	}
```

 * The next two sections also need a few things (Server Name & .crt + .key path)

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

* Build your own Nginx container. Do this from the LDAPDockerRegistry directory
``` bash
# the dot at the end of the line matters. Don't forget it.
$ cd nginx && docker build --tag=ldap_nginx .
```

* Start the containers. This assumes you got docker-compose in your $PATH. Once more, Execute this command from inside the LDAPDockerRegistry directory
``` bash
docker-compose up
```

* Add user in your LDAP server. The LDAP object your create **MUST be of the 'Person' type** if you haven't edit the LDAP connection string in the Nginx configuration.
To do this, multiple options : 
	- LDAP utilities (go search for them with your favorite packet manager) and do a ldapadd
	- Some UI. You can quickly setup one with the following (edit the variables to suit to your needs) : 
``` bash
$ docker run -p 80:80 -p 443:443 -e LDAP_HOST=<LOCAL LDAP IP ADDRESS> -e LDAP_BASE_DN=dc=your,dc=domain,dc=com -e LDAP_LOGIN_DN=cn=admin,dc=your,dc=domain,dc=com -d windfisch/phpldapadmin
# And then, go check it with your browser at port 80 or 443.
```

## A word about self-signed certificate 
Docker doesn't like self-signed certificate. Really. <return> If you choose to use a self-signed certificate, you're probably got some troubles when trying to connect to your private registry. You need to add the --insecure-registry to your DOCKER_OPTS. More on that [here](https://docs.docker.com/registry/insecure/) for most of the Linux distros. <return> For some others (like CentOS 7), you can edit the Docker service definition (search for it with a find command or something). Edit the ExecStart line : 
``` bash
ExecStart=/usr/bin/docker daemon --insecure-registry <IP ADDRESS OF YOUR NGINX PROXY> -H fd://
```

# Testing
Go ahead and log into your brand new and beautiful Docker registry with his magnificent Nginx + LDAP backend. 
``` bash
$ docker login <NGINX HOST>  # Do not provide any email address. And try fake credentials, just to be sure...
...
Login Succeeded
```

You can now push/pull images to your own private registry. Congrats. 
``` bash
$ docker pull hello-world # grab some lightweight image
$ docker tag hello-world <YOUR HOST>/hw 
$ docker push <YOUR HOST>/hw
```
You should now be able to pull this image from another logged-in docker host. 

You can also grab UIs to explore your registry, like [this one](https://hub.docker.com/r/atcol/docker-registry-ui/) or [this one](https://hub.docker.com/r/hyper/docker-registry-web/). 

Happy pulls and pushes :)