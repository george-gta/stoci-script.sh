#!/bin/bash
!/bin/bash
# Enables a mode of the shell where all executed commands are printed to the terminal.

set -x

##Created: 12.06.2019
##Maintainer: Stoyan Petkov <stoyan@gotoadmins.com>
##Purpose: This script will request a new cert from let's encrypt and add it to existing server
##         This script is a wrapper around acme.sh (https://github.com/Neilpang/acme.sh) with custom additions
##           We don't need to re-invent the wheel.


#variables
acme_base="/root/.acme.sh/"
nginx_base="/etc/nginx/ssl"
nginx_conf="/etc/nginx/sites-enabled"
acme_bin="/root/.acme.sh/acme.sh"
wwwroot="/var/www/html"

# If the following variable is empty, then echo "Please provide action add | remove"

if [[ -z $1 ]]; then
  echo "Please provide action add | remove"
  exit 1
fi
# action equals to the variable $1
action=$1

# If the following variable is empty, then echo "Please provide domain name"
if [[ -z $2 ]]; then
  echo "Please provide domain name "
  exit 1
fi
# domain equals to the variable $2
domain=$2

case $action in
  add)
        # If acme ( Automated Certificate Management Environment ) is installed, then install directly ( without asking for a confirmation ) socat (The socat command shuffles data between two locations) and then install .acme.sh/ in the /root directory (acme.sh requires and it's recommended socat to be installed on the machine
        #check if we have acme installed
        if [ ! -f ${acme_bin} ]; then
          apt-get install -y socat
          curl https://get.acme.sh | sh
        fi

        # Makes parent directory within the /etc/nginx directory and archives the following variables with the cp -a command (because of the -a extension), then checks and tests the nginx.conf file and restarts the nginx service
        #issue cert
        ${acme_bin} --issue -d ${domain} -w ${wwwroot}
        mkdir -p /etc/nginx/ssl
        cp -a ${acme_base}/${domain}/ ${nginx_base}

       	${acme_bin} --install-cert \
                  -d ${domain} \
                  --cert-file \
                  ${nginx_base}/${domain}/fullchain.cer \
                  --key-file ${nginx_base}/${domain}/${domain}.key \
                  --fullchain-file ${nginx_base}/${domain}/fullchain.cer \
                  --reloadcmd "nginx -t && service nginx restart"

        #merge certificates

        # EOF ( The end marker helps the computer know it has allocated enough space to store the file)
        if [ ! -f ${nginx_conf}/${domain} ]; then
            cat > ${nginx_conf}/${domain} <<-EOF

server {
   listen 80;
   server_name  ${domain} ;
   return 301 https://$server_name$request_uri;
}


server {
  listen 443 ssl http2;
  server_name ${domain};
  ssl on;
  ssl_certificate ${nginx_base}/${domain}/fullchain.cer;
  ssl_certificate_key ${nginx_base}/${domain}/${domain}.key;
  ssl_session_timeout 5m;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;
}
EOF

        fi
       # nginx -t (checks and tests the nginx.conf file) and if $? is true, then restart the nginx service, then removes ${nginx_conf}/${domain} and ${nginx_base}/${domain} and if the test of the nginx.conf file is successfull, then restarts nginx.service. Every other action leads to a message "Invalid action add|remove only"
        nginx -t
        if [ $? -eq 0 ]; then
          service nginx restart
        fi

 ;;
 remove)
    rm -fv ${nginx_conf}/${domain} || exit 1
    rm -rf ${nginx_base}/${domain}
    if $( nginx -t ); then
      service nginx restart
    fi

  ;;
  *)
    echo "Invalid action add|remove only"
    exit 1
esac

exit 0
