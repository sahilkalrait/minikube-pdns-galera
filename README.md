This repository contains a working configuration for PowerDNS setup on Kubernetes with Mysql backend.
MariaDB galera cluster is used as backend. 

We use [severalnines](https://github.com/severalnines) setup of MariaDB Galera cluster:
https://github.com/severalnines/galera-docker-mariadb/tree/master/example-kubernetes

**Only Development Version**:

This installation is Tested only on a **minikube** cluster.
 
So, it can only be used in development environment.

It requires following steps (not exhaustive) to reach a production state:
1. Testing on a multi-node cluster
2. Add https support
3. Add secrets for credentials stored in plaintext

**Tools Installed and Access**:
1. `PowerDNS`: A deployment for PowerDNS service (API disabled). Served on `<minikube_node_ip>:30053`
2. `PowerDNS API`: A deployment for PowerDNS API service. Served on `pdns-api.minikube:80`
3. `PowerDNS Admin`: A deployment for [PowerDNS Admin GUI](https://github.com/ngoduykhanh/PowerDNS-Admin). Served on `pdns-admin.minikube:80`
4. `Nginx Ingress`: Ingress controller to expose the PowerDNS API and PowerDNS Admin on http
5. `Mariadb 10.1 Galera`: A MariaDB galera cluster from [severalnines](https://github.com/severalnines/galera-docker-mariadb)

**Directory structure**:
1. `nginx-ingress-controller`: Contains k8s configuration files for **nginx-ingress-controller**
2. `pdns/galera`: Contains configuration files to setup a **Galera cluster**
3. `pdns`: Contains configuration files to setup **PDNS server, PDNS API server and PDNS-Admin server**


**Installation Steps**:
1. Install **minikube** on local and start a cluster: https://kubernetes.io/docs/tasks/tools/install-minikube/
2. Add `/etc/hosts` entry to <`minikube_node_ip`> for following domains:
    1. `pdns.minikube`
    2. `pdns-api.minikube`
    3. `pdns-admin.minikube`
2. Install **nginx-ingress-controller** in "ingress-nginx" namespace: 
    1. Create Controller: `kubectl apply -f nginx-ingress-controller/`
    2. Create Ingress Rules: `kubectl apply -f pdns/pdns_nginx_ingress.yaml`
3. Install **Mariadb 10.1 Galera cluster** in "galera_cluster" namespace: 
    1. Create Namespace: `kubectl apply -f pdns/galera/galera_namespace.yaml`
    2. Create etcd cluster for galera: `kubectl apply -f pdns/galera/galera_etcd.yaml`
    3. Change replica to 1 in Deployment in file `pdns/galera/galera_mariadb.yaml`
    4. Deploy 1 node galera cluster: `kubectl apply -f pdns/galera/galera_mariadb.yaml`
    5. Change replica to 3 in Deployment in file `pdns/galera/galera_mariadb.yaml`
    6. Scale the cluster to size 3: `kubectl apply -f pdns/galera/galera_mariadb.yaml`
4. Install **pdns** in "default" namespace:
    1. Create Config map for pdns: `kubectl create configmap pdns-conf --from-file=pdns.conf=pdns.conf`
    2. Deploy the pdns: `kubectl apply -f pdns/pdns_deployment.yaml`
    3. Create SSH tunnel to access mysql on local mysql client: 
        `ssh -i ~/.minikube/machines/minikube/id_rsa -NfL 5000:<galera_db_service_ip>:3306 docker@<minikube_node_ip>`
    3. Install pdns schema in mysql
        `mysql -h 127.0.0.1 -P 5000 -u pdns -p pdns < pdns/pdns_schema.sql`
    4. Access webserver on `pdns.minikube`
    5. Access DNS on `pdns.minikube:30053`
5. Install **pdns-api** in "default" namespace:
    1. Create Config map for pdns-api: `kubectl create configmap pdns-api-conf --from-file=pdns-api.conf=pdns-api.conf`
    2. Deploy the pdns-api: `kubectl apply -f pdns/pdns_api_deployment.yaml`
    3. Access API on `pdns-api.minikube:80`
6. Install **[PowerDNS Admin GUI](https://github.com/ngoduykhanh/PowerDNS-Admin)** in "default" namespace:
    1. Login to Mysql as "root" user. Create database with name `powerdns_admin` and provide all privileges to "pdns" user
    2. Create Config map: `kubectl create configmap pdns-admin-conf --from-file=config.py=pdns_admin_config.py`
    3. Deploy the GUI: `kubectl apply -f pdns/pdns_admin_deployment.yaml`
    3. Access GUI on `pdns-admin.minikube:80`



