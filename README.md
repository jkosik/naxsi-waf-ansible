# To customize
Put waf-ssl.crt and waf-ssl.key into files/etc/nginx/ssl so that nginx can do https. waf-ssl.key should use Ansible Vault.

# Deployment of naxsi

Naxsi is a web application firewall (WAF) based on nginx. Ansible playbook creates standalone WAF in reverse proxy setup with local Elasticsearch database for storing logs, mainly for Learning purposes.

Architecture follows Naxsi principles as showed in [Naxsi documentation](https://raw.githubusercontent.com/wiki/nbs-system/naxsi/Images/naxsi-workflow.png) with exception:
* Naxsi and Nginx logs go to central Graylog (no Kibana used)
* During Learning phase naxsi.log goes also to local Elasticsearch database on WAF from where the whitelists are being created

# Playbook
**To customize playbook it is needed to check global_vars.**

### Running the playbook
```
$ git clone <this repo>
$ ansible-playbook -i inventory main.yml --ask-vault-pass
```

### inventory file
```
[naxsi]
xx.xx.xx.xx inv_hostname="naxsi-test"
```
### Required roles
* ansible-py2-bootstrap
* ansible-fqdn
* ansible-set-proxy

# Naxsi Usage
https://github.com/nbs-system/naxsi/tree/master/nxapi


### nginx restart
```
nginx -s stop
nginx
```

### Switching between learning and normal mode

(Un)comment LearningMode; in /etc/nginx/sites-available/nginx-naxsi.

### Run Elasticsearch and test
```
service elasticsearch start
curl -X GET 'http://xx.xx.xx.xx:9200'
curl -XGET 'http://xx.xx.xx.xx:9200/_nodes?pretty'
```

# Learning Phase
### Create index in ES
```
curl -XPUT 'http://xx.xx.xx.xx:9200/nxapi/'
```

### Import data do ES from naxsi.log after learning took place
Do not run more times on the same input to avoid duplicit data in Elasticsearch DB.
```
cd /home/ubuntu/nginx-naxsi/naxsi-master/nxapi/
./nxtool.py -c nxapi.json --files=/var/log/nginx/naxsi.log
```

### Check imported data
```
curl -XPOST "http://xx.xx.xx.xx:9200/nxapi/events/_search?pretty" -d '{}'
```

### Get data summary
```
cd /home/ubuntu/nginx-naxsi/naxsi-master/nxapi/
./nxtool.py -c nxapi.json  -x --colors
```

### Create whitelist (-f brings additional features for granularity)
Whitelist is being created from naxsi.log, not nginx access.log. Only logs intercepted by naxsi as potential issues (_naxsi naxsi.log 2017/01/27 08:30:36 [error] 11110#0: *27 NAXSI\_FMT:..._) are used for whitelist suggestions.
```
cd /home/ubuntu/nginx-naxsi/naxsi-master/nxapi/
./nxtool.py -c nxapi.json -f
./nxtool.py -c nxapi.json  -f --filter 'uri search' --slack
```
If --slack is present, scoring is ignored, and all possible whitelists are displayed.
If --slack is present, scoring is ignored, and all possible whitelists are displayed. In normal conditions, whitelists with more than 0 points are displayed. 
**Pick up and insert requested whitelist rules into _naxsi\_whitelist.rules_ and refer to the file in nginx sites-available file.**

Reference to whitelist file must be in _location_ section of the sites file
```
server {
    location / {
        include /etc/nginx/naxsi_whitelist.rules;
    {
}
```
_Turn off learning in nginx-naxsi sites-available file._

### Tagging events in local ES dababase
Used for ignoring those events ES, which were already pushed to whitelists and it is not needed to analyse them again.
```
cd /home/ubuntu/nginx-naxsi/naxsi-master/nxapi/
./nxtool.py -c nxapi.json -s xx.xx.xx.xx -w /etc/nginx/naxsi_whitelist.rules --tag
```
Example: I created whitelist containing everything identified during Learning (expecting automated rapid learning approach in trusted environment). Once I tag all logs imported in ES before using this whitelsit, I will see empty output when checking ES again. No additional whitelist would be able to be created.
```
./nxtool.py -c nxapi.json -x --colors# size :1000
# Whitelist(ing) ratio :
# Top servers :
# Top URI(s) :
# Top Zone(s) :
# Top Peer(s) :
```
I would need additional naxsi.log import to something new there.

