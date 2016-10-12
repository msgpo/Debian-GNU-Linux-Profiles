##SSL/TLS Mutual Authentication in Gitlab

##### Copyright (c) TYA
##### Homepage: http://tya.company/


##### Gitlab Project Homepage: http://www.gitlab.com
###### Note: We using Gitlab Community Edition, Installation Step in copy from Offcial Installation.


#####Update and Install Dependencies
```
apt-get update
apt-get install curl openssh-server ca-certificates postfix
```

#####Add the GitLab package server and install the package
```
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
apt-get install gitlab-ce
```

#####Disable built-in Nginx
```
#In /etc/gitlab/gitlab.rb

# Disable the built-in nginx
nginx['enable'] = false
```

#####Reload Gitlab Services
```
gitlab-ctl restart
```

#####Configure Nginx
```
cat >/etc/nginx/sites-available/gitlab-ssl<EOF
## GitLab
##
## Modified from nginx http version
## Modified from http://blog.phusion.nl/2012/04/21/tutorial-setting-up-gitlab-on-debian-6/
## Modified from https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html
## Modified from https://gitlab.com/gitlab-org/gitlab-ce/blob/master/lib/support/nginx/gitlab-ssl
##
## Lines starting with two hashes (##) are comments with information.
## Lines starting with one hash (#) are configuration parameters that can be uncommented.
##
##################################
##        CONTRIBUTING          ##
##################################
##
## If you change this file in a Merge Request, please also create
## a Merge Request on https://gitlab.com/gitlab-org/omnibus-gitlab/merge_requests
##
###################################
##         configuration         ##
###################################
##
## See installation.md#using-https for additional HTTPS configuration details.

upstream gitlab-workhorse {
  server unix:/var/opt/gitlab/gitlab-workhorse/socket fail_timeout=0;
}

## Redirects all HTTP traffic to the HTTPS host
server {
  ## Either remove "default_server" from the listen line below,
  ## or delete the /etc/nginx/sites-enabled/default file. This will cause gitlab
  ## to be served if you visit any address that your server responds to, eg.
  ## the ip address of the server (http://x.x.x.x/)
  listen 0.0.0.0:80;
  listen [::]:80 ipv6only=on default_server;
  server_name YOUR_SERVER_FQDN; ## Replace this with something like gitlab.example.com
  server_tokens off; ## Don't show the nginx version number, a security best practice
  return 301 https://$http_host$request_uri;
  access_log  /var/log/nginx/gitlab_access.log;
  error_log   /var/log/nginx/gitlab_error.log;
}

## HTTPS host
server {
  listen 0.0.0.0:443 ssl;
  listen [::]:443 ipv6only=on ssl default_server;
  server_name YOUR_SERVER_FQDN; ## Replace this with something like gitlab.example.com
  server_tokens off; ## Don't show the nginx version number, a security best practice

  ## Strong SSL Security
  ## https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html & https://cipherli.st/
  ssl on;
  #example:
  #                   +-------+
  #                   |Root CA|
  #                   +-------+
  #                     |
  #                     |
  #              +---------------+
  #              | Gitlab Sub-CA |
  #              +---------------+
  #                |     |     |
  #                |     |     |
  #              USER1 USER2 USERn
  ssl_certificate ${PATH to your certificate chain};
  ## cat server-cert.pem intermediate-ca.pem root-ca.pem >cert-bundle.pem
  ssl_certificate_key ${PATH to your gitlab server key};
  ssl_client_certificate ${PATH to your certificate chain};
  ssl_verify_client on;
  ssl_verify_depth 2;

    if ($ssl_client_i_dn != "/CN=xxx Company GitLab Certificate Authority/OU=xxx Infrastructure Assurance/O=xxx Company/C=CN") {
        return 403;
	## We use x.509 attribute to restrict access.
    }
  # GitLab needs backwards compatible ciphers to retain compatibility with Java IDEs
  #ssl_ciphers "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4";
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;
  ssl_session_timeout 5m;

  ## See app/controllers/application_controller.rb for headers set

  ## [Optional] If your certficate has OCSP, enable OCSP stapling to reduce the overhead and latency of running SSL.
  ## Replace with your ssl_trusted_certificate. For more info see:
  ## - https://medium.com/devops-programming/4445f4862461
  ## - https://www.ruby-forum.com/topic/4419319
  ## - https://www.digitalocean.com/community/tutorials/how-to-configure-ocsp-stapling-on-apache-and-nginx
  # ssl_stapling on;
  # ssl_stapling_verify on;
  # ssl_trusted_certificate /etc/nginx/ssl/stapling.trusted.crt;
  # resolver 208.67.222.222 208.67.222.220 valid=300s; # Can change to your DNS resolver if desired
  # resolver_timeout 5s;

  ## [Optional] Generate a stronger DHE parameter:
  ##   sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 4096
  ##
  # ssl_dhparam /etc/ssl/certs/dhparam.pem;

  ## Individual nginx logs for this GitLab vhost
  access_log  /var/log/nginx/gitlab_access.log;
  error_log   /var/log/nginx/gitlab_error.log;

  location / {
    client_max_body_size 0;
    gzip off;

    ## https://github.com/gitlabhq/gitlabhq/issues/694
    ## Some requests take more than 30 seconds.
    proxy_read_timeout      300;
    proxy_connect_timeout   300;
    proxy_redirect          off;

    proxy_http_version 1.1;

    proxy_set_header    Host                $http_host;
    proxy_set_header    X-Real-IP           $remote_addr;
    proxy_set_header    X-Forwarded-Ssl     on;
    proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
    proxy_set_header    X-Forwarded-Proto   $scheme;
    proxy_pass http://127.0.0.1:8080;
  }
  error_page 404 /404.html;
  error_page 422 /422.html;
  error_page 500 /500.html;
  error_page 502 /502.html;
  error_page 503 /503.html;
  location ~ ^/(404|422|500|502|503)\.html$ {
    root /home/git/gitlab/public;
    internal;
  }
}

EOF
```
#####Reload Nginx Services
```
service nginx reload
```

#####Connection diagram
```
+--------------+   Local Socket   +------------+   SSL Mutual Authentication   +----+
|Gitlab Service|------------------|Nginx Server|-------------------------------|User|
+--------------+                  +------------+                               +----+
```


######Reference: 
######[1] https://about.gitlab.com/downloads/#debian8
######[2] http://docs.gitlab.com/omnibus/settings/nginx.html