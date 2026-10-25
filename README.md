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

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

We can throw it up real quick and login using default credentials just to test that it's up.
```bash
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
```bash
docker run --rm -ti wazuh/wazuh-indexer:4.14.9 bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
```

Copy the hash and insert it into the admin hash section of the `internal_users.yml`. 

Access the container with `docker exec -it <INDEXER_CONTAINER_NAME> bash`

Set the following environmental variables
```bash
export INSTALLATION_DIR=/usr/share/wazuh-indexer
export CONFIG_DIR=$INSTALLATION_DIR/config
CACERT=$CONFIG_DIR/certs/root-ca.pem
KEY=$CONFIG_DIR/certs/admin-key.pem
CERT=$CONFIG_DIR/certs/admin.pem
export JAVA_HOME=/usr/share/wazuh-indexer/jdk
```

After a couple minutes you can run the `securityadmin.sh` script to apply the password change.
```bash
bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh -cd $CONFIG_DIR/opensearch-security/ -nhnv -cacert  $CACERT -cert $CERT -key $KEY -p 9200 -icl
```
Doing this one at a time, we can change the passwords of all internal users.

# Exercises
## Cracking SSH
I used `hydra` to brute force the SSH password of a local target box (VM) on my home network. 

First, I installed a Wazuh agent on the target box
<img width="914" height="962" alt="Screenshot_20261024_163310" src="https://github.com/user-attachments/assets/c9812662-6d93-450e-80fa-f309a191e9a2" />

<img width="936" height="927" alt="Screenshot_20261024_163605" src="https://github.com/user-attachments/assets/77d5c3a4-c191-43d6-8748-937a9f299d12" />

We can verify that the agent was installed on the agents screen.
<img width="922" height="287" alt="Screenshot_20261024_164214" src="https://github.com/user-attachments/assets/560c474d-f181-4416-8250-a964c1ac0aeb" />

Then I used `hydra` with the `rockyou.txt` wordlist to attempt to crack the target boxes SSH password
(Note, I used -t 4 flag after this screenshot so that it wasn't so aggressive)
<img width="859" height="299" alt="Screenshot_20261024_164444" src="https://github.com/user-attachments/assets/f207a1bf-d1fe-44b2-b55e-fe80cfc3cd6d" />

On the threat hunting screen, we can see the results of the `hydra` attack. We can see from the MITRE ATTACKS pie chart that all of the attacks were SSH, password guessing, and brute force

<img width="934" height="1016" alt="Screenshot_20261024_164631" src="https://github.com/user-attachments/assets/5d402594-8d35-4eb2-b8d7-e2b2520ae3b9" />

## File Integrity Monitoring
I configured the Wazuh agent to monitor two directories in real time by adding the following lines in `/var/osssec/etc/ossec.conf` on the target box. Don't forget to restart the service after.
```
  <directories realtime="yes">/etc</directories>
  <directories realtime="yes">/home/labuser</directories>
```

First, i created, modified, and removed a test file to set if the integrity monitoring was active
<img width="731" height="171" alt="Screenshot_20261025_141606" src="https://github.com/user-attachments/assets/bce588ba-f696-4c89-8380-14597ada37ff" />

We can see the events in the logs, and even look at the hash digests between modifications
<img width="1918" height="612" alt="Screenshot_20261025_141933" src="https://github.com/user-attachments/assets/c80da6d8-fffc-4aa1-a8e2-2a20a07036ee" />
<img width="745" height="225" alt="modify_diff" src="https://github.com/user-attachments/assets/fc8dda88-3b88-4916-817f-8e18d4d1f35e" />

### Editing /etc/shadow and /etc/passwd
An example of what would actually be worth considering escalating is a change in a vulnerable file. For this example I'm using `/etc/shadow` and `/etc/passwd`

I created a test user, a password for the test user, and deleted the test user.
```bash
sudo useradd -m test
sudo passwd test
sudo userdel -r test
```

We can see when a user was created, how many bytes were added to the file, the diff, and who did the change (root).
<img width="725" height="366" alt="adduserpasswd" src="https://github.com/user-attachments/assets/35492d77-b5e5-4f85-8412-0527f6c4ea7e" />

We can decide if this is worth escalation depending on the UID/GID range and group membership of the new user
```bash
sudo id test
uid=1001(test) gid=1001(test) groups=1001(test)
```

We can confirm that this user has no sudo/wheel membership, and a non-zero system UID. We can say that this is a normal account rather than a privilege escalation attempt. If the UID was 0, or it was a UID from an existing account, then we can escalate and take action.
