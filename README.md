# Cableguard

# What it is
Cableguard is a set of tools that make it easy to deploy and manage Wireguard 
servers and clients at scale. The **cableguard_api** allows you to create 
servers, assign address ranges and then dynamically assign addresses from that 
range to clients. It can output data in JSON as well as Wireguard configuration 
file format and allows working with 'clusters' (groups of servers that share the 
same clients). You can use this stand-alone and implement further dissemination 
of the configurations through your network yourself. Or you can use the rest of 
the tools to provision automatically. 

Watch a video of the tool in action here: https://www.youtube.com/watch?v=2ERNDbl1VAM

With our **cableguard_provisioner** Ansible playbook it is trivial to launch and 
configure new Wireguard servers which automatically connect to the API to 
download their configurations and are dynamically updated when clients are added
or removed through the API, without the need to reload Wireguard.

The Ansible Playbook registers your server with Cableguard and exchanges some keys 
for further secure communications. After registration the Wireguard server only talks 
to the configuration deliverer (**cableguard_cd** ) which is a stripped down version 
of the API which *only* handles sending out the configurations. 
You can have this publicly accessible while keeping the full  **cableguard_api** 
firewalled from the public Internet.

# How it works
**cableguard_api**
* Use JSON (or the web interface) to define servers and clusters (groups of servers with the same client). Specify the network range for the server or cluster (e.g. 10.0.0.0/16)
* Add clients to the server or cluster and have an IP address assigned to them automatically
* Scan the QR code or Wireguard Config from the web interface or use the JSON response to configure the client... Or use `cableguard_cd` and `cableguard_provisioner` for automated provisioning, see below:

**cableguard_cd**
* Start the cableguard_cd instance and make sure it can communicate with:
	* the database used by `cableguard_api`
	* the machine that runs the Ansible playbook
	* the Wireguard servers that are configured through the Ansible playbook

**cableguard_provisioner**
* Take note of the `server token` value that is generated and stored with a server you've created in the API
* Launch a new blank Ubuntu machine at your favorite cloud provider (or use your own)
* Provide the `server token` to the Ansible Playbook through `deploy.sh`, along with the cableguard_api server, the Wireguard target server and the username (note that you can use any sudoable user)
* The Ansible Playbook will now install Wireguard, configure it for IP forwarding, register itself with `cableguard_api`, install SSL certificates for further communication and exchange SSH keys

**cableguard_docker**
The **cableguard_docker** repository contains scripts that setup **cableguard_api** and **cableguard_cd** all in one server with SSL certs and database, ready to use. It is most likely the easiest way to get started. 

# Quick installation
1. Make sure you have docker and docker-compose installed.
2. Clone the cableguard_docker repository to the server that will function as your provisioning server: `git clone https://github.com/keezel-co/cableguard_docker`
3. Execute the following commands:
```
cd cableguard_docker
./init_submodules.sh
./run.sh
```

Docker will now download and create all the necessary containers. When everything is ready you will find the following ready for you:

https://yourserver:1443/ - the API and web interface (running with self-signed certificate)

https://yourserver:2443/ - the Register API

https://yourserver:3443/ - The Cableguard Configuration Deliverer

# Installation
**cableguard_api**
1. Ensure Docker and `docker-compose` are installed on your system
2. Clone the `cableguard_api` repository `git clone https://github.com/keezel-co/cableguard_api/`
3. Modify the `config.py` file with a connection string to your database. You can run `create_database.py` after doing this to setup all the tables.
4. Start the docker container	`cd cableguard_api && docker-compose up --build`

**cableguard_cd**
1. Ensure Docker and `docker-compose` are installed on your system
2. Clone the `cableguard_cd` repository `git clone https://github.com/keezel-co/cableguard_cd/`
3. Modify the `config.py` file with a connection string to your database. Ensure this points to the same database as **cableguard_api**
4. Start the docker container	`cd cableguard_cd && docker-compose up --build`

**cableguard_provisioner**
1. Clone the `cableguard_provisioner` repository `git clone https://github.com/keezel-co/cableguard_provisioner/`
2. Make sure you know a `server token` for the Wireguard server you wish to provision and execute `deploy.sh` as follows: `./deploy.sh -t 38041963-df28-40e6-b203-1c6c7e4134c0 -w cableguard.example.com -s wireguard.example.com -u root` where after `-u` goes the user the Ansible script can login as. Obviously, this requires you to be able to login to the address specified by `-w` through ssh. Note that you can use any sudoable user.

If you use `-h` the following help will be displayed:
```
Usage: ./deploy.sh <params> where params can be:
       -h To display this help and exit
       -t Token to use
       -w Cableguard hostname from where configuration is downloaded
       -s Address of wireguard server to provision
  Optional:
       -p Cableguard port (default 2443)
       -u Remote user of wireguard's SSH (root or sudoable user) (default root)
```

# Security
The Cableguard server does not have any user level security and is meant to be ran on internal infrastructure of ISPs. It should be ran on private networks and/or firewalled to limit access.
There is a separate package, `cableguard-cd` meant to be run on public IPs where Wireguard servers can register and fetch their configuration from. This package only includes the `GetServerConfig` and `GetClusterConfig` calls and checks if the request is made with the appropriate `token`

# Data storage
If you just want to run Cableguard on a single server or on your Raspberry Pi at home, you can use the included SQLite database. Otherwise, you can change the `SQLALCHEMY_DATABASE_URI` to connect to your database.
If you can successfully connect to your database you can run the included `create_database.py` script to automatically generate all the necessary tables for you.

# API 

The easiest way to check out how the API works is to simply open up Developer Tools in your browser and inspect the calls being made through the web interface. Below you will find some examples of what JSON to post to:

**Create a server**

Post to */api/servers/add*

```
    {"server_description":"Your description here",
    "server_country":"Netherlands",
    "server_city":"Amsterdam",
    "server_ip":"12.21.12.21",
    "server_networkv4":"10.0.0.0/24",
    "server_networkv6":"2001:db00::0/24",
    "server_port":"12345",
    "server_dns":"8.8.8.8",
    "server_pubkey":null,
    "server_privkey":null,
    "server_postup":null,
    "server_postdown":null,
    "server_persistentkeepalive":"21",
    "cluster_id":null}
```
Note that `server_pubkey` and `server_privkey` will be automatically generated and stored if not provided.

**Delete a server**

DELETE request `/api/servers/delete/<int:server_id>`

**Add a client to a server**

POST request to `/api/clients/add_to_server/<int:server_id>`

```
    {"server_id":"1",
    "client_description":"Macbook"}
```
Note that the `client_privkey` is returned once in response to this call *but not stored*. Make sure you store this value if you need to re-use it later.

**Delete a client**

DELETE request `/api/clients/delete/<int:client_id>`

**Get a list of all servers**

GET request `api/servers/get/all`

**Get a list of all clients for a server**

GET request `/api/clients/get/by_server_id/<int:server_id>`
