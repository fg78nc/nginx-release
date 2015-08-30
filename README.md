# BOSH-deployed Nginx Server

This BOSH release can be used to deploy an nginx server.

### Procedure for creating the BOSH manifest

Copy a manifest from the examples subdirectory. If you're not using SSL, copy *nginx-aws.yml*. If you're using SSL, copy *nginx-ssl-aws.yml*.

```bash
cp examples/nginx-aws.yml ~
```
Edit your BOSH manifest. Search for occurrences of "CHANGEME" and substitute your values as appropriate.

```bash
vim ~/nginx-aws.yml
```

While editing, remember to populate the *nginx* job's configuration (i.e. `nginx.conf`). For example,

```yaml
jobs:
- name: nginx
  properties:
    nginx_conf: |
      worker_processes  1;
      error_log /var/vcap/sys/log/nginx/error.log   info;
      #pid        logs/nginx.pid; # PIDFILE is configured via monit's ctl
      events {
        worker_connections  1024;
      }
      http {
        include /var/vcap/packages/nginx-1.6.2/conf/mime.types;
        default_type  application/octet-stream;
        sendfile        on;
        keepalive_timeout  65;
        server_names_hash_bucket_size 64;
        server {
          listen 80;
          access_log /var/vcap/sys/log/nginx/sslip.io-access.log;
          error_log /var/vcap/sys/log/nginx/sslip.io-error.log;
        }
      }
```

### Deploy

```bash
bosh-init deploy ~/nginx-aws.yml
```

### HTML content

You must manually add the HTML content *after* successful deployment (irritating, we know)

First, set your environment variables. You'll need the elastic IP of the deployed VM
and the key pair to ssh in. Both these items should be in the BOSH manifest:

```bash
export ELASTIC_IP=52.0.76.229 # substitute your VM's elastic IP
export AWS_KEY_PAIR=~/.ssh/aws_nono.pem # substitute the path to your key pair
```

Next, we copy the *document_root* directory onto the VM. The document root
should have the file index.html (e.g. *document_root/index.html*)

```bash
# copy the files to the nginx VM's /tmp/ directory
scp -r -i $AWS_KEY_PAIR document_root vcap@$ELASTIC_IP:/tmp/
```

We assume that in the BOSH manifest that the nginx.conf specifies
*/var/vcap/jobs/nginx/document_root* as the root. We move our
uploaded file into place:

```bash
# ssh in and become root
ssh -i $AWS_KEY_PAIR vcap@$ELASTIC_IP
sudo su - # password is 'c1oudc0w'
mv /tmp/document_root /var/vcap/jobs/nginx/
```

Point your browser to your VM's elastic IP and make sure that the page is the one
you expect.

### Addendum: BOSH Jobs

There are two jobs:

* nginx: runs the nginx server
* fetcher: periodically fetches [computationally] expensive pages for the nginx
  server to server as static pages. Discussion of this job is beyond the scope
  of this document and it can be safely ignored (i.e. you don't need to instantiate
  the job in the BOSH manifest).

### Caveats

* We've only tested on AWS
* works with Ubuntu and CentOS stemcell (uses `apt-get` to install PCRE pre-requisite on Ubuntu)
