## Create SSL Certificates

Step 1:
The first step is creating a self-signed certificate. Though other tools can be used, Apache NiFi Toolkit is used in this article because it generates everything you need with a single command. Download the latest Apache NiFi Toolkit from the official website.

[APACHE NIFI TOOLKIT](https://nifi.apache.org/download.html)

Step 2:
Extract the archived file, and change the terminal directory to that folder.

```bash
tar -xvzf nifi-toolkit-1.15.3-bin.tar.gz
cd nifi-toolkit-1.15.3
```

COPY

Step 3:
Run the following command to generate the SSL certificates and the necessary configurations.

```bash
./bin/tls-toolkit.sh standalone -n localhost -C 'CN=admin,OU=NiFi' --subjectAlternativeNames 'localhost,0.0.0.0,nifiserver1,nifiserver2,nifiserver3,nifiserver4'
```

COPY

The above command generates the certificate, key, keystore, truststore, and the properties file for a NiFi server deployed in localhost. Note that the Subject Alternative Names have four servers. These names will be the hostnames of our hardcoded Apache NiFi containers in Step 1.3. Therefore add as many hostnames as possible based on your cluster size.

```treeview
├── localhost
│   ├── keystore.jks
│   ├── nifi.properties
│   └── truststore.jks
├── nifi-cert.pem
├── nifi-key.key
└── CN=admin_OU=NiFi.p12
```

COPY

## Create A NiFi Cluster Using Docker Compose

Though you can use plain Docker commands, Docker compose is used here to make the instructions clear. In addition, you can also add additional services in the same Docker compose configuration.

Step 1:
Create a new folder named `nifi` anywhere you want.

```bash
mkdir ~/nifi
cd ~/nifi
```

COPY

Step 2:
Copy the keystore.jks, truststore.jks, nifi-cert.pem, and nifi-key.key in to this folder.

```bash
cp $NIFI_TOOLKIT_HOME/localhost/keystore.jks ./
cp $NIFI_TOOLKIT_HOME/localhost/truststore.jks ./
cp $NIFI_TOOLKIT_HOME/localhost/nifi-cert.pem ./
cp $NIFI_TOOLKIT_HOME/localhost/nifi-key.key ./
```

COPY

Step 3:
Create a new file named `docker-compose.yml` with the following content:

```haml
version: "3"
services:
    zookeeper:
        hostname: zookeeper
        container_name: zookeeper
        image: 'zookeeper:latest'
        ports:
            - 2181
        environment:
            - ALLOW_ANONYMOUS_LOGIN=yes

    nifi_1:
        image: apache/nifi:1.15.0
        container_name: nifi_1
        hostname: nifiserver1
        restart: unless-stopped
        ports:
            - 8443
        depends_on:
            - zookeeper
        environment:
            - NIFI_WEB_HTTPS_PORT=8443
            - SINGLE_USER_CREDENTIALS_USERNAME=admin
            - SINGLE_USER_CREDENTIALS_PASSWORD=ctsBtRBKHRAx69EqUghvvgEvjnaLjFEB
            - NIFI_SENSITIVE_PROPS_KEY=rHkWR1gDNW3R
            - NIFI_WEB_PROXY_HOST=
            - NIFI_WEB_HTTPS_HOST=
            - NIFI_CLUSTER_ADDRESS=
            - NIFI_REMOTE_INPUT_HOST=
            - AUTH=tls
            - KEYSTORE_PATH=/opt/certs/keystore.jks
            - KEYSTORE_TYPE=JKS
            - KEYSTORE_PASSWORD=GpHqcDbGc6/VQO2Y5z4tWpDMKHAXa4JvaxL6 lQwYKc
            - TRUSTSTORE_PATH=/opt/certs/truststore.jks
            - TRUSTSTORE_TYPE=JKS
            - TRUSTSTORE_PASSWORD=Nf6rMSYb9G6Zw2Idwwz/LUGYNmgBTo0Xf6D86CNbvuU
            - NIFI_SECURITY_USER_AUTHORIZER=single-user-authorizer
            - NIFI_SECURITY_USER_LOGIN_IDENTITY_PROVIDER=single-user-provider
            - NIFI_CLUSTER_NODE_PROTOCOL_PORT=8082
            - NIFI_ZK_CONNECT_STRING=zookeeper:2181
            - NIFI_ELECTION_MAX_WAIT=1 min
            - NIFI_CLUSTER_IS_NODE=true
        volumes:
            - ./keystore.jks:/opt/certs/keystore.jks
            - ./truststore.jks:/opt/certs/truststore.jks

    nifi_2:
        image: apache/nifi:1.15.0
        container_name: nifi_2
        hostname: nifiserver2
        restart: unless-stopped
        ports:
            - 8443
        depends_on:
            - zookeeper
        environment:
            - NIFI_WEB_HTTPS_PORT=8443
            - SINGLE_USER_CREDENTIALS_USERNAME=admin
            - SINGLE_USER_CREDENTIALS_PASSWORD=ctsBtRBKHRAx69EqUghvvgEvjnaLjFEB
            - NIFI_SENSITIVE_PROPS_KEY=rHkWR1gDNW3R
            - NIFI_WEB_PROXY_HOST=
            - NIFI_WEB_HTTPS_HOST=
            - NIFI_CLUSTER_ADDRESS=
            - NIFI_REMOTE_INPUT_HOST=
            - AUTH=tls
            - KEYSTORE_PATH=/opt/certs/keystore.jks
            - KEYSTORE_TYPE=JKS
            - KEYSTORE_PASSWORD=GpHqcDbGc6/VQO2Y5z4tWpDMKHAXa4JvaxL6 lQwYKc
            - TRUSTSTORE_PATH=/opt/certs/truststore.jks
            - TRUSTSTORE_TYPE=JKS
            - TRUSTSTORE_PASSWORD=Nf6rMSYb9G6Zw2Idwwz/LUGYNmgBTo0Xf6D86CNbvuU
            - NIFI_SECURITY_USER_AUTHORIZER=single-user-authorizer
            - NIFI_SECURITY_USER_LOGIN_IDENTITY_PROVIDER=single-user-provider
            - NIFI_CLUSTER_NODE_PROTOCOL_PORT=8082
            - NIFI_ZK_CONNECT_STRING=zookeeper:2181
            - NIFI_ELECTION_MAX_WAIT=1 min
            - NIFI_CLUSTER_IS_NODE=true
        volumes:
            - ./keystore.jks:/opt/certs/keystore.jks
            - ./truststore.jks:/opt/certs/truststore.jks

    nginx:
        image: nginx:latest
        volumes:
            - ./nginx.conf:/etc/nginx/nginx.conf:ro
            - ./nifi-cert.pem:/etc/nginx/ssl/nifi-cert.pem:ro
            - ./nifi-key.key:/etc/nginx/ssl/nifi-key.key:ro
        depends_on:
            - nifi_1
            - nifi_2
        ports:
            - "8443:8443"
```

COPY

Note that the NiFi server is configured with a single username (`admin`) and password (`ctsBtRBKHRAx69EqUghvvgEvjnaLjFEB`). Running the above configuration will create two NiFi containers using the keystore and truststore generated in Step 1.3. Both NiFi services have the same configuration except for the hostname and container\_name. The hostname must match the `subjectAlternativeNames` provided in Step 1.3 and each container must have a unique hostname. Nginx is used to load balance the requests to the NiFi UI.

Zookeeper is used for leader election in the NiFi cluster and the Nginx is used to provide a single endpoint for the NiFi dashboard. Nginx also sets the header to localhost (See Step 5) to satisfy the NiFi hostname verification.

Step 4:
In the above configuration, random `KEYSTORE_PASSWORD` and `TRUSTSTORE_PASSWORD` are used. Open the `nifi.properties` file generated in Step 1.3 and search for `nifi.security.keystorePasswd` and `nifi.security.truststorePasswd`. Use the value of `nifi.security.keystorePasswd` as the value of `KEYSTORE_PASSWORD` and the value of `nifi.security.truststorePasswd` as the value of `TRUSTSTORE_PASSWORD`.

Step 5:
Create a new file named `nginx.conf` with the following content. The Nginx is configured with HTTPS using the same certificates used for Apache NiFi. Note the header name is set to localhost because NiFi will verify the hostname in the request with the hostname defined in the SSL certificate. If you have additional NiFi containers, add them to the upstream configuration in the “nginx.conf”.

```javascript
user nginx;

events {
    worker_connections 1000;
}
http {

    upstream nifi_cluster {
        server nifi_1:8443;
        server nifi_2:8443;
        ip_hash;
    }

    server {
        listen 8443;
        ssl on;
        ssl_certificate /etc/nginx/ssl/nifi-cert.pem;
        ssl_certificate_key /etc/nginx/ssl/nifi-key.key;
        location / {
            proxy_set_header Host "localhost";
            proxy_pass https://nifi_cluster;
        }
    }
}
```

COPY

## Run The NiFi Cluster in Docker with SSL

Step 1:
Open a terminal in the “nifi” folder and run the following command. Please note that you must have the Docker Compose installed.

```bash
docker-compose up
```

COPY

Step 2:
Wait for some time for the NiFi to boot up and then visit [http](https://localhost:8443/nifi)[s://loc](https://localhost:8443/nifi)[alhost:8443/nifi](https://localhost:8443/nifi) from your browser. You will see a warning page because we are using a self-signed certificate.


Step 3:
Accept the risk and visit the page. Use the username: `admin` and password: `ctsBtRBKHRAx69EqUghvvgEvjnaLjFEB` as defined in the docker-compose.yml to login.

Step 4:
Check the “Cluster” option from the hamburger menu. You must see the NiFi containers defined in the docker-compose.yml as the members of the NiFi cluster.
