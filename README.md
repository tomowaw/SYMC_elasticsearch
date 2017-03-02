# Introduction
Everything here is made by me and for the soul purpose of learning. This is still work in progress and there is absolutely no warranty that it will work. In fact it will not work almost certainly but if you are willing to give it a try then ill gladly help you.
You can contact me on lech.lachowicz@gmailthisshouldberemoved.com


# A short howto install
Install Centos 7 minimal and set up interfaces

Set the password that will be used later to access kibana
```
export KIBANAADMIN_PASSWD="topsecretpassword"
```

#Install all required dependencies
```
yum -y install vim wget mailx tcpdump
cd ~
wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u73-b02/jdk-8u73-linux-x64.rpm"

sudo yum -y localinstall jdk-8u73-linux-x64.rpm

rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

Disable SELinux
```
cat << EOF > /etc/selinux/config
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
EOF
```

Add Elasticsearch repo:
```
cat << EOF > /etc/yum.repos.d/elastic.repo
[elasticsearch-5.x]
name=Elasticsearch repository for 5.x packages
baseurl=https://artifacts.elastic.co/packages/5.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```

And install Elasticsearch, Kibana and logstash
`yum -y install elasticsearch logstash kibana`

Configure Elasticsearch by setting it up to listen on localhost
```
cat << EOF >> /etc/elasticsearch/elasticsearch.yml
network.host: "localhost"
script.engine.groovy.inline.aggs: true
script.engine.groovy.inline.update: on
EOF
```
Start Elasticsearch
```
systemctl start elasticsearch
systemctl enable elasticsearch
```

Configure Kibana
```
echo "server.host: \"localhost\"" >> /etc/kibana/kibana.yml
```

And start it
```
systemctl start kibana
systemctl enable kibana
```

Now lets go for nginx:
```
yum -y install epel-release
yum -y install nginx httpd-tools

# set the password for kibanaadmin
htpasswd -b -c /etc/nginx/htpasswd.users kibanaadmin "$KIBANAADMIN_PASSWD"

# setup nginx
cat << EOF > /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '\$remote_addr - \$remote_user [\$time_local] "\$request" '
                      '\$status $body_bytes_sent "\$http_referer" '
                      '"\$http_user_agent" "\$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;
}
EOF

cat << EOF > /etc/nginx/conf.d/kibana.conf
server {
    listen 80;

    server_name example.com;

    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/htpasswd.users;

    location / {
        proxy_pass http://localhost:5601;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host \$host;
        proxy_cache_bypass \$http_upgrade;        
    }
}
EOF
```

And start nginx

```
systemctl start nginx
systemctl enable nginx
```


Download and install SMG logstash configs
```
cd /etc/logstash/conf.d/
wget https://raw.githubusercontent.com/ponquersohn/SYMC_elasticsearch/master/conf.d/smg.conf
mkdir /etc/logstash/conf.d/patterns
cd patterns
wget https://raw.githubusercontent.com/ponquersohn/SYMC_elasticsearch/master/conf.d/patterns/smg
```

Restart logstash
```
systemctl restart logstash
systemctl status logstash
systemctl enable logstash
```


Dont forget to install missing firewall rules.
```
firewall-cmd --add-port=80/tcp
firewall-cmd --add-port=80/tcp --permanent

firewall-cmd --add-port=5140/tcp
firewall-cmd --add-port=5140/tcp --permanent
```

As of now logstash is waiting for syslog logs on port 5140, parses everything and loads it to elasticsearch. 
You can access kibana under http://elk.host 

# Remember to sync time between ELK host and SMG!!!





