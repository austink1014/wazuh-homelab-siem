# wazuh-homelab-siem
A writeup of my experience with Wazuh in my personal homelab.

Local IPs and anything else I find personal is redacted.

## Wazuh Setup
### Prerequisites
I decided I wanted to set up a single-node stack using Docker.

The reason being is the host machine is a bare Debian install, and I didn't want to add more repositories.
Wazuh provides a clean, containerized way to deploy its SIEM solution without worrying about dependencies breaking.

To get started, I had to configure the Linux kernel, `max_map_count` is used by the Linux kernel to determine the maximum amount of virtual memory maps . The default `max_map_count` is 65530. I needed to increase the maximum amount of virtual memory areas (VMAs) to *262144* because the Wazuh indexer creates a lot of VMAs. 

To make this a persistent change on my host machine, I added the following line to `/etc/sysctl.d/99-wazuh-max-map.conf`

```
vm.max_map_count=262144
```

Before deployment, though I wanted to check if any of the ports in my homelab would conflict with Wazuh. Wazuh uses ports `1514`,`1515`,`514`,`55000`,`9200`, and `443`.

To do this I used the `ss` command, a utility useful for dumping socket statistics.

```
sudo ss -tulpn
```
<img width="2184" height="275" alt="image" src="https://github.com/user-attachments/assets/c85ae7b6-52a9-439f-9051-d94fd3e35b5b" />

- t - displays TCP

- u - displays UDP

- l - only listening sockets

- p - shows name and PID of each socket (this is the one that requires sudo privilege)

- n - show numeric ports and addresses

Seeing this, none of the ports currently deployed in my server conflict with Wazuh's ports, so there is no need to redirect any ports (thankfully wouldn't be hard in docker-compose)

### Deployment
There is an easily deployable docker repository at 
```
https://github.com/wazuh/wazuh-docker
```
I cloned the repository using `git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.9` and then changed directory to the single-node folder `cd wazuh-docker/single-node/`

Each node needs a signed x509 certificate, the `wazuh-certs-generator.yml` file provides a convenient docker image for generating Wazuh self-signed certificates. The certificates are stored in `wazuh-docker/single-node/config/wazuh_indexer_ssl_certs`

```
docker compose -f generate-indexer-certs.yml run --rm generator
```

We can throw it up real quick and login using default credentials just to test that it's up.
```
sudo docker compose up -d
```
We can check if they are up and running using `docker ps`
<img width="2404" height="263" alt="image" src="https://github.com/user-attachments/assets/763b5026-47b0-4d8e-9955-6b7ae96b0542" />

Connecting with `https://<SERVER-HOST-IP>`, we get a scary warning from the browser!
<img width="954" height="843" alt="679482291-660c8293-e2e4-4e14-83c2-8ef1c04ab8b2" src="https://github.com/user-attachments/assets/e313913f-fecc-4f9c-a95c-6fae10cae004" />

Just kidding, It's because the certificates we made aren't signed by a browser-trusted certificate authority, so we can just get past this by clicking advanced and proceed. 

We get a really pretty login screen, we can use the defaults to log in for now, we will change this soon!

Username: `admin`

Password: `SecretPassword`

<img width="954" height="843" alt="image" src="https://github.com/user-attachments/assets/0fff9a59-2b3a-4bd5-9999-48e0a8ce8779" />

### Changing the admin Password
Log out, stop the stack, and generate the hash

To change the password, we need to put our passwords in plaintext in the docker-compose.yml, the manager and indexer log in as admin, so they need the un-hashed password.

In the uploaded docker-compose.example.yml, these fields have `<PLEASE-CHANGE-ME>`

To generate a hash for our new password, we can use the provided bcrypt hash function in the docker container
```
docker run --rm -ti wazuh/wazuh-indexer:4.14.9 bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
```

Copy the hash and insert it into the admin hash section of the `internal_users.yml`. 

Access the container with `docker exec -it <CONTAINER_NAME> bash`

Set the following environmental variables
```
export INSTALLATION_DIR=/usr/share/wazuh-indexer
export CONFIG_DIR=$INSTALLATION_DIR/config
CACERT=$CONFIG_DIR/certs/root-ca.pem
KEY=$CONFIG_DIR/certs/admin-key.pem
CERT=$CONFIG_DIR/certs/admin.pem
export JAVA_HOME=/usr/share/wazuh-indexer/jdk
```

After a couple minutes you can run the `securityadmin.sh` script to apply the password change.
```
bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh -cd $CONFIG_DIR/opensearch-security/ -nhnv -cacert  $CACERT -cert $CERT -key $KEY -p 9200 -icl
```
Doing this one at a time, we can change the passwords of all internal users.
