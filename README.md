# Deploying Highly Available K3S with External Database

## Introduction

In this blog, I will explain how to deploy K3S in HA configuration with an external database postgres. [K3S](https://k3s.io/) is a certified kubernetes distribution for IoT and Edge computing. I deployed it on virtual machines in a IBM Z mainframe. Instead of etcd, I choose postgres as my storage for my K3S clusters. I deployed postgres in non HA mode. For information on how to deploy postgres in HA mode go though their [official documention](https://www.postgresql.org/docs/13/high-availability.html). The communication between postgres and the K3S cluster is SSL secured. I used nginx at layer 4 in front of my K3S servers for load balancing. This is because that we may add new server nodes or a server might go down. So, instead of the K3S server IP's, if we use the loadbalancer, we don't face such issues and also achieve HA.

## HA Configuration

The following image describes how I deployed my K3S cluster with two nodes in HA mode. Even if one server goes down, then the other server is still reachable and the load balancer routes all requests to the other one. Since all components of K3S are stateless even if a server goes down, we don't loose any information. All information is stored in postgres and it must be HA.

![K3S HA Congiguration](K3S-HA.png)

## Environment

You need the following environment before you get started. 

Operating System - SLE15SP2
Virtual Machines - 5
VCPU             - 2
RAM              - 8 GB
Disk             - 30 GB
Arch             - s390x

The following are my IP addresses for different virtual machines. I will be using these IP  addresses in this blog and you can always lookup here for what the IP address represents.

K3s Server 1 -  `10.161.129.54`
K3S Server 2 -  `10.161.129.154`
K3S Agent    -  `10.161.129.196`
Postgres     -  `10.161.129.212`
Nginx        -  `10.161.129.118`

I am deploying in mainframe (s390x) virtual machines but the same commands can be used for any other architecture. Now, lets get to the commands to install postgres and K3S cluster.

## Installation of Postgres

I installed postgres10 in one of the virtual machines. Let's name the virtual machine as postgres. Copy paste the below commands to install postgres. I ensured that there is mutual TLS communication between the postgres database and the K3S clusters for security. I have used self-signed certificates for identifying postgres which were created using openssl.

### Postgres Virtual Machine

* Install postgres package using zypper

```
zypper -n in postgresql10 postgresql10-server
systemctl start postgresql
```

* Create K3S database, user role and grant all access to the user role

```
sudo -u postgres psql
create database k3s;
create user k3s with encrypted password 'k3s';
grant all privileges on database k3s to k3s;
exit;
```

We will be using `k3s` database for storing the cluster information. We will use the user `k3s` for K3S cluster to authenticate with the postgres database.

* Create self-signed certificates which identify the postgres server and store them in `/var/lib/pgsql/data/` 

```
openssl req -new -x509 -days 365 -nodes -text -out /var/lib/pgsql/data/postgres.crt   -keyout /var/lib/pgsql/data/postgres.key -subj "/CN=postgres.rancher.rke2" -addext "subjectAltName=DNS:postgres.rancher.rke2" 
```

* Ensure limited access to the private key otherwise postgres server would complain about it.

```
chmod 0600 /var/lib/pgsql/data/postgres.key
chown postgres:postgres /var/lib/pgsql/data/postgres.key
```

* Copy the public key certificate identifying postgres to the two K3S servers so that the K3S servers can verify the identity of postgres for SSL communication.

```
scp /var/lib/pgsql/data/postgres.crt sles@10.161.129.54:
scp /var/lib/pgsql/data/postgres.crt sles@10.161.129.154:
```

* Replace the contents of the `/var/lib/pgsql/data/pg_hba.conf` with the following content.

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections onlyf
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            ident
hostssl all             all             0.0.0.0/0               md5 clientcert=verify-full
```

The contents of the file say that for localhost connections there is no need of password or ssl communication to connect to the database. But for all other connections to the postgres, communication must happen only through ssl with client verification (K3S server is the client for postgres) and password authentication.

Now, before we proceed further in setting up postgres database, we shall create certificates identifying the K3S cluster and copy the certificate to postgres virtual machine so that postgres can verify the identity of K3S server.

### K3S Server 1 Virtual Machine

* Create a self signed certificate identifying K3S cluster and give the right permissions to the private key

```
openssl req -new -x509 -days 365 -nodes -text -out k3s.crt -keyout k3s.key -subj "/CN=k3s" -addext "subjectAltName=DNS:k3s" 
chmod 0600 k3s.key
```

* Copy the public key certificate to the postgres machine so that postgres can verify the K3S client.

```
scp /home/sles/k3s.crt sles@10.161.129.212:
```

* Copy the public and private keys to the other K3S server. The two servers together represent the K3S cluster.

```
scp /home/sles/k3s.crt /home/sles/k3s.key sles@10.161.129.154:
```

Now, lets shift back to our postgres virtual machine.

### Postgres Virtual Machine

* Move the k3s.crt into `/var/lib/pgsql/data` directory so that postgres configuration file could use it.

```
mv /home/sles/k3s.crt /var/lib/pgsql/data/
```

* Modify the contents of `/var/lib/pgsql/data/postgresql.conf` with the following values

```
listen_addresses = '*'
ssl = on
#ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL' # allowed SSL ciphers
#ssl_prefer_server_ciphers = on
#ssl_ecdh_curve = 'prime256v1'
#ssl_dh_params_file = ''
ssl_cert_file = '/var/lib/pgsql/data/postgres.crt'
ssl_key_file = '/var/lib/pgsql/data/postgres.key'
ssl_ca_file = '/var/lib/pgsql/data/k3s.crt'
```

`listen_addresses` : should be `*` or the IP of your postgres server. This makes sure that postgres server is listening on the node's IP address. 

`ssl` : Turn on the ssl so that communication only happens in a secured way.
`ssl_cert_file` `ssl_key_file` : These certificates identify the postgres database. We already created them at the start of the blog. Now, we just point them to the cert locations. 

`ssl_ca_file` : This is a CA (Certificate Authority) certificate which identifies the clients of postgres. In our example, K3S is the client. So, I created a self signed certificate for K3S and pointed out `ssl_ca_file` to the self signed public certificate of K3S cluster.

* Restart postgres server to adopt the new configuration

```
systemctl restart postgresql
```

Now that we successfully deployed our postgres database, lets shift to K3S virtual machine and install K3S there.

### K3S Server 1 Virtual Machine

* Install K3S server with the right flags and values.

```
curl -sfL https://get.k3s.io | sh -s - server --datastore-endpoint="postgres://k3s:k3s@postgres.rancher.rke2:5432/k3s" --datastore-cafile="/home/sles/postgres.crt" --token=k3s --datastore-certfile="/home/sles/k3s.crt" --datastore-keyfile="/home/sles/k3s.key" --tls-san=10.161.129.118
```

`--datastore-endpoint` : the format is `postgres://username:password@hostname:port/database-name` for postgres. In my case, I created a role `k3s` with password `k3s` and the database name is also `k3s`. For the hostname, I put a name `postgres.rancher.rke` since the certs were created with the CN value as this name. 

`--datastore-cafile` : Put the public key certificate of postgres so that K3S can verify the identity of postgres using this certificate. In a self-signed certificate, the public certificate acts as the CA authority and can verify itself.

`--datastore-certifle` : this is the public certificate which identifies the K3S cluster.

`--datastore-keyifle` : this is the private key which belongs to the K3S cluster.

`--token` : a secret passsword is to be created so that other servers or agents can connect to this K3S cluster.

`--tls-san` : this is the load balacner IP address.

* For K3S to resolve `postgres.rancher.rke2` I appended my `/etc/hosts` file with the following 

```
10.161.129.212  postgres.rancher.rke2
```

where `10.161.129.212` is the ip address of the postgres server.

* Lets verify if our K3S server is running properly and is connected to the postgres or not 

```
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get pods -A
```

You should see a list of pods with all in `running` status. If not, you could use 

```
journalctl -xe
```

to look for errors that occurred while installing K3S server.

### K3S Server 2 Virtual Machine

* Install the 2nd K3S server with the same command as used for installing the first server

```
curl -sfL https://get.k3s.io | sh -s - server --datastore-endpoint="postgres://k3s:k3s@postgres.rancher.rke2:5432/k3s" --datastore-cafile="/home/sles/postgres.crt" --token=k3s --datastore-certfile="/home/sles/k3s.crt" --datastore-keyfile="/home/sles/k3s.key" --tls-san=10.161.129.118
```

* Verify if K3S server is running properly and is connected to postgres or not

```
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get pods -A
```

You should see something like this 

```
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   svclb-traefik-55frd                       2/2     Running   0          11m
kube-system   svclb-traefik-x59vc                       2/2     Running   0          2m43s
kube-system   local-path-provisioner-6c79684f77-55tkc   1/1     Running   0          107s
kube-system   coredns-d76bd69b-5n8s7                    1/1     Running   0          107s
kube-system   traefik-df4ff85d6-88phx                   1/1     Running   0          107s
kube-system   metrics-server-7cd5fcb6b7-x7t2r           1/1     Running   0          107s
```

* You could also check if the cluster has two servers and if they have the `master` role by running this command

```
kubectl get nodes
```

You should see something like this

```
NAME           STATUS   ROLES                  AGE   VERSION
k3s-server-1   Ready    control-plane,master   14m   v1.23.6+k3s1
k3s-server-2   Ready    control-plane,master   29s   v1.23.6+k3s1
```

Now, before connecting a K3S agent where we K3S schedules its workload, we need to add a load balancer in front of our servers so that users or K3S agents can talk to it.

### Nginx Load Balancer Virtual Machine

I used nginx in front of the K3S cluster server nodes at layer 4 of the network stack. We are forwarding all the requests that come at port 6443 to the load balancer to one of k3S servers. The kubernetes API server listens to that port.

* Install nginx package

```
zypper in nginx
```

* Create a file `/etc/nginx/nginx.conf` and fill in with the contents below

```
load_module /usr/lib64/nginx/modules/ngx_stream_module.so;

worker_processes 4;
worker_rlimit_nofile 40000;


events {
    worker_connections 8192;
}

stream {
    log_format logs '$remote_addr - - [$time_local] $protocol $status $bytes_sent $bytes_received $session_time "$upstream_addr"';

    access_log /var/log/nginx/access.log logs;

    upstream k3s_api_server {
        least_conn;
        server 10.161.129.54:6443 max_fails=3 fail_timeout=5s;
        server 10.161.129.154:6443 max_fails=3 fail_timeout=5s;
    }
    server {
        listen 6443;
        proxy_pass k3s_api_server;
    }
}
```

We are using the `least_conn` algorithm to decide which K3S server should the request go to. Nginx will route the request to the server which has the least active connections according to this algorithm.

* Start nginx for these changes to take effect

```
nginx -s reload
systemctl reload nginx && systemctl restart nginx
```

Now, we setup our load balancer and so anyone can now talk to our K3S servers. Lets now add a K3S agent which talks to this load balancer and registers itself.

### K3S Agent Virtual Machine

* Install K3S agent using the following command

`--server` : mention the loadbalancer ip address

```
curl -sfL https://get.k3s.io | sh -s - agent --token=k3s --server https://10.161.129.118:6443
```

### K3S Server 1 Virtual Machine

* Now, you could check if the K3S agent is successfully registered by running this command

```
kubectl get nodes
```

You should see something like this

```
NAME           STATUS   ROLES                  AGE    VERSION
k3s-server-1   Ready    control-plane,master   41m    v1.23.6+k3s1
k3s-server-2   Ready    control-plane,master   28m    v1.23.6+k3s1
k3s-agent      Ready    <none>                 105s   v1.23.6+k3s1
```


So, we have successfully installed postgres, K3S servers and a K3S agent. You now a highly available K3S cluster with an external database. Note that full HA is only achieved if postgres is also deployed in HA configuration.

# References 
- K3S Architechture Docs - https://rancher.com/docs/k3s/latest/en/architecture/#high-availability-with-an-external-db 
- K3S Datastore Docs -
https://rancher.com/docs/k3s/latest/en/installation/datastore/
- K3s installation Docs -
https://rancher.com/docs/k3s/latest/en/installation/ha/
- Nginx Load Balancer Docs - 
https://rancher.com/docs/rancher/v2.5/en/installation/resources/k8s-tutorials/infrastructure-tutorials/nginx/
  
