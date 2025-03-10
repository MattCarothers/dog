<p align="center">
  <img src="../../images/dog-segmented-green.network-200x200.png">
</p>

<h1>dog_trainer</h1>

dog_trainer is the central server component of [dog](https://github.com/relaypro-open/dog),
a centralized firewall management system.

- [Runtime Dependencies](#runtime-dependencies)
- [Build Dependencies](#build-dependencies)
- [CA Certificate Creation](#ca-certificate-creation)
- [RethinkDB setup](#rethinkdb-setup)
- [RabbitMQ setup](#rabbitmq-setup)
- [App setup](#app-setup)
- [Build Release Deploy](#build-release-deploy)
- [Run](#run)
- [Logs](#logs)
- [Deploy Agents](#deploy-agents)

## Runtime Dependencies

- linux 4.x+ (Ubuntu 16.04 tested)
- diffutils
- coreutils
- rabbitmq-server 3.7+
- rethinkdb 2.3.6+
- git

## Build Dependencies

- erlang 22+

### CA Certificate Creation

If you already have a CA and per-server certs, you can reuse them, or buy new ones
(costly).
You can create your own self-signed certs with your own Certificate Authority.

Examples Ansible scripts are provided to get you started.  Examples use Credstash
to securely store the CA files [credstash](https://github.com/fugue/credstash)

#### Create CA

Create the CA and store it in a secure location,
 use it in a deletable location like a tmpfs or an encrypted filesystem and then
 delete after use.

- example:

```bash
scripts/ansible/dog_create_ca_cert.yml
```

#### Create server cert

One TLS cert must be created for each dog_trainer and dog_agent server

- example:

```bash
scripts/ansible/setup_dog_cert.yml
```

### RethinkDB setup

#### Install RethinkDB

```bash
sudo apt install rethinkdb=2.3.6~0xenial
```

#### Setup clustering

For high availability, setup replication: [sharding-and-replication](https://rethinkdb.com/docs/sharding-and-replication/)

#### RethinkDB security

RethinkDB's web console doesn't require authentication,
but you can use oauth2_proxy [oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy)
for that purpose.

#### Create database tables and indexes

```bash
#edit src/rethink_db_setup.erl
$ ./rebar3 shell
```

```erlang
1> rethink_db_setup:setup_rethinkdb($ENV_NAME).
2> dog_ipset:set_hash(dog_ipset:create_hash(dog_ipset:create_ipsets())).
```

#### Import default Services

Definitions for some well known services can be imported.

```bash
rethinkdb import -f /opt/dog_trainer/scripts/default_services.json --table dog.service
```

### RabbitMQ setup

#### install RabbitMQ

      sudo apt install rabbitmq-server=3.7.17-1

#### Copy and edit config

```bash
cp config/rabbitmq/rabbitmq.conf /etc/rabbitmq/
#edit rabbitmq to reflect location of certs and 
#you should replace PRIVATE_IP with an IP that is not accessible via the
#pubilc internet.
cp config/rabbitmq/enabled_plugins /etc/rabbitmq/
restart rabbitmq-server
```

#### Create virtual host and admin users

```bash
#edit setup_rabbitmq.sh, define variables.
script/setup_rabbitmq.sh
```

### Networking

#### Create DNS service names

It's useful to create service names for the dog_trainer clients to connect to.  
You may want to create one for each type of connection:

- rabbitmq clients: To force rabbitmq connections over private networks in EC2,
create this service name as a A Record with the private IP of your server as its
value.
- dog_park (gui) clients

#### Federate multiple brokers to connect to multiple regions

You can deploy multiple rabbitmq servers across regions, using the federation
plugin to replcate client queues
and exchanges.  You can then connect your agents to these distributed rabbitmqs
via their local private IPs,
avoiding having to have agents connect to a rabbitmq in another region across
the public internet.
[RabbitMQ federation](https://www.rabbitmq.com/federation.html)  

### App setup

Create directories:

```bash
./install.sh
```

If credentials stored in credstash, run to fill templates:

```bash
cd dog_trainer/template_setup
$ ../rebar3 shell
1> template_setup:main().
#otherwise copy config/templates/*.dtl to config/ and manually update credentials.
```

## Build Release Deploy

```bash
$ rebar as $ENV tar

#copy tar to system, extract to /opt/dog_trainer
```

## Run

Setup systemd service

```bash
cp config/dog_trainer.service /lib/systemd/system/dog_trainer.service
systemctl enable dog_trainer
systemctl start dog_trainer
```

## Logs

```bash
/var/log/dog_trainer/
```

## Deploy Agents

  [Agents](https://github.com/relaypro-open/dog_agent)
