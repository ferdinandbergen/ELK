# Elasticsearch + kibana + Logstash + Nginx on Ubuntu 18.04

Co-Created with [@bennwitt](https://github.com/bennwitt)  

New changes are that we added Logstash, SAMBA, and NGINX


## How To Install Elasticsearch and Kibana on Ubuntu Linux


## Install Elasticsearch

1. Elastic signs all of its packages with the Elasticsearch PGP Signing Key. Add this key to your server.

```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
```

2. As the packages will be retrieved via HTTPS, you will need to install the apt-transport-https package.

```
sudo apt install apt-transport-https
```

3. You will next need to add the Elastic repository.

There are different repositories for the standard distribution (X-Pack pre-bundled) and the Apache 2.0 licensed distribution. You will use the standard distribution in order to take advantage of the Security features of the X-Pack Basic tier.

```
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
```

4. You will now need to update the local cache of available packages and their versions.

```
sudo apt update
```

5. You can now install Elasticsearch.

```
sudo apt install elasticsearch
```

6. Configure JVM Heap.

Java Virtual Machine is included with Elastic 7.x, so for Elastic and Kibana, there is no need to download Java.  (Logstash will however require it, so we handle that later). If a JVM is started with unequal initial and max heap sizes, it may pause as the JVM heap is resized during system usage. For this reason it’s best to start the JVM with the initial and maximum heap sizes set to equal values.  This file can also be found in the /etc/ folder

Edit `/etc/elasticsearch/jvm.options` and set `-Xms` and `-Xmx` to about one third of the system memory, but do not exceed `31g`. For example...

```
# Xms represents the initial size of total heap space
# Xmx represents the maximum size of total heap space

-Xms8g
-Xmx8g
```

7. Increase System Limits.

You should specify system limits in a systemd configuration file for the elasticsearch service.  This file is included in the /etc/ folder

```
sudo mkdir /etc/systemd/system/elasticsearch.service.d
```

Copy the provided file `etc/systemd/system/elasticsearch.service.d/elasticsearch.conf` to `/etc/systemd/system/elasticsearch.service.d`

Additionally, copy the provided file `etc/sysctl.d/70-elasticsearch.conf` to `/etc/sysctl.d`. Reboot the system for these changes to take effect.

8. Modify the Elasticsearch configuration.

Replace the default Elasticsearch configuration file with the provided configuration by copying `etc/elasticsearch/elasticsearch.yml` to `/etc/elasticsearch`

Modify this configuration as may be appropriate for your environment.

9. Start Elasticsearch.

```
sudo systemctl daemon-reload
```

```
sudo systemctl start elasticsearch.service
```

10. Enable Elasticsearch to start automatically when the system is started.

```
sudo systemctl enable elasticsearch.service
```

11. Set passwords for the Elasticsaerch, Kibana, Logstash, Filebeats application:

```
sudo /usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```

12. Test those passwords
Browse to the Elasticsearch Application from browser http://server.company.com:9200.  when prompted, enter in "elastic" then the passwrod you entered in for Step #11

13. Test a query on Elasticsearch
```
curl -X GET "localhost:9200/?pretty"
```


## Install Kibana

1. Install Kibana.

```
sudo apt install kibana
```

2. Modify the Kibana configuration.

Replace the default Kibana configuration file with the provided configuration by copying `etc/kibana/kibana.yml` to `/etc/kibana`


3. Starting Kibana the Very first time:

please be patient, to load the kibana.yml config we just copied into the Kibana, Kibana's first load must be "optimized" for the installation.

start kibana to "test" the config and if test is successful, optimize Kibana
```
sudo /usr/hare/kibana/bin/kibana --allow-root
```
PLEASE BE PATIENT:  this may take 10 minutes to load the first time. 


4. Start Kibana.

the step above, Kibana is running from the CMD line, not as a service.  Load Kibana as a service below.

```
sudo systemctl daemon-reload
```

```
sudo systemctl start kibana.service
```

Be patient. It may take a moment the first time as Kibana optimizes enabled applications.

5. Enable Kibana to start automatically when the system is started.

```
sudo systemctl enable kibana.service
```

## NGINX Proxy

1. Install nginx proxy so Kibana uses port 80 instead of 5601
```
apt install nginx apache2-utils -y
```

2. COnfigure Nginx

open and edit /etc/nginx/sites-available/kibana
Insert this:

```
server {
	listen 80;

	server_name elk_server;
	server_name elk_server.company.net;

	location / {
		proxy_pass http://localhost:5601;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection 'upgrade';
		proxy_set_header Host $host;
		proxy_cache_bypass $http_upgrade;
	}
}
```

3. enable this new Kibana profile for nginx
```
ln -s /etc/nginx/sites-available/kibana /etc/nginx/sites-enabled/
```
and test the config
```
nginx -t
```


4. Enable NGinx to run as a service
```
systemctl enable nginx
```
and restart nginx
```
systemctl restart nginx
```




## Install Logstash

1. Install Java
```
apt install default-jre
```

2. edit /etc/profile
add this text to the end of the document

```
JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
PATH=$PATH:$HOME/bin:$JAVA_HOME/bin
export JAVA_HOME
export JRE_HOME
export PATH
```

3. Set JAVA_HOME
```
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```
verify java home
```
echo $JAVA_HOME
```


4. Install Logstash
```
apt-get update && sudo apt-get install logstash
```



## Install the common componets

1.
```
apt install software-properties-common
```



## Install SAMBA

We use SAMBA to drop files onto the ELK server directly.   to do this, we need SAMBA

1. install samaba
```
apt-get install samba -y
```

2. mkdir of the shared folder and give everyone full access to that folder.
```
mkdir -m777 inbound
```

3. create local SAMBA user
```
smbpasswd -a {{sambauser}}
```
password
```
changeme
```

4. Change SMB config
```
nano /etc/samba/smb.conf
```

change these lines
	
	workgroup = {{elkgroup}}
	
	server string = %h

change message body for "[printers]" to:
```
[inbound]
	comment = inbound
	browseable = yes
	path = /index/inbound/
	guest ok = yes
	ready only = no
	create mask = 0777
```
remove next print section altogether

5. restart SMB to that the sahre is active
```
/etc/init.d/smbd restart
```

6. Test SMB storage location
go to WIndows machine that test that \\elk_server\inbound\ can be reached and if its available.




## TESTING
1. create a symbolic link for logstash to run in your folder
```ln -s /usr/share/logstash/bin/logstash logstash```

2. run it from the command line
```./logstash -w 1 -f /index/inbound/config/maincookie.conf```
