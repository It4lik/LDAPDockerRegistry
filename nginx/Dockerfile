FROM centos:7

MAINTAINER Nick Halm <nhalm@hfmi.com>

ENV NGINX_VERSION release-1.9.5

RUN yum update -y \
	&& yum groupinstall -y "Development Tools"  \
	&& yum install -y epel-release \
	&& yum install -y  git \
	openssl-devel \
	openldap-devel \
	pcre-devel \ 
	ca-certificates \
	gcc \
	make

# See http://wiki.nginx.org/InstallOptions
RUN mkdir /var/log/nginx \
	&& mkdir /etc/nginx \
	&& cd ~ \
	&& git clone https://github.com/kvspb/nginx-auth-ldap.git \
	&& git clone https://github.com/nginx/nginx.git \
	&& cd ~/nginx \
	&& git checkout tags/${NGINX_VERSION} \
	&& ./auto/configure \
		--add-module=/root/nginx-auth-ldap \
		--with-http_ssl_module \
		--with-debug \
		--conf-path=/etc/nginx/nginx.conf \ 
		--sbin-path=/usr/sbin/nginx \ 
		--pid-path=/var/log/nginx/nginx.pid \ 
		--error-log-path=/var/log/nginx/error.log \ 
		--http-log-path=/var/log/nginx/access.log \ 
	&& make install \
	&& cd .. \
	&& rm -rf nginx-auth-ldap \
	&& rm -rf nginx 

EXPOSE 80 443

CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
