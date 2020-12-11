# Deploying MySQL with Adminer on Akash DeCloud
![main](https://github.com/yuravorobei/mysql-on-akash-guide/blob/main/media/main.png)

In this guide, I'll show you how to deploy a MySQL database server on Akash decentralized cloud along with the Adminer utility for management databases via a web interface.

**Akash** is a marketplace for cloud compute resources which is designed to reduce waste, thereby cutting costs for consumers and increasing revenue for providers. Akash DeCloud is a faster, better, and lower cost cloud built for DeFi, decentralized projects, and high growth companies, providing unprecedented scale, flexibility, and price performance. 10x lower in cost, Akash serverless computing platform is compatible with all cloud providers and all applications that run on the cloud.

All actions require you to have some knowledge to work with the Linux console. All actions I perform on a Ubuntu 18.04 operating system. And so we go.

# Installing and preparing Akash
To interact with the Akash system, you need to install it and create a wallet. I will follow the official [Akash Documentation guides](https://docs.akash.network/v/master/).

### Choosing a Network
At any given time, there are a number of different active Akash networks running, each with a different akash version, chain-id, seed hosts, etc… Three networks  are currently available: mainnet, testnet and edgenet. In this guide for the "Akashian Challenge: Phase 3 Start Guide" i'll use the `edgenet` network.

So let’s define the required shell variables:
  ```bash
  $ AKASH_NET="https://raw.githubusercontent.com/ovrclk/net/master/edgenet"
  $ AKASH_VERSION="$(curl -s "$AKASH_NET/version.txt")"
  $ AKASH_CHAIN_ID="$(curl -s "$AKASH_NET/chain-id.txt")"
  $ AKASH_NODE="$(curl -s "$AKASH_NET/rpc-nodes.txt" | shuf -n 1)"
  ```

### Installing the Akash client
There are several ways to install Akash: from the [release page](https://github.com/ovrclk/akash/releases), from the source or via `godownloader`. As for me, the easiest and the most convenient way is to download via `godownloader`:
  ```bash
  $ curl https://raw.githubusercontent.com/ovrclk/akash/master/godownloader.sh | sh -s -- "$AKASH_VERSION"
  ```
And add the akash binaries to your shell PATH. Keep in mind that this method will only work in this terminal instance and after restarting your computer or terminal this PATH addition will be removed. You can read how to set your PATH permanently [on this page](https://stackoverflow.com/questions/14637979/how-to-permanently-set-path-on-linux-unix) if you want:
  ```bash
  $ export PATH="$PATH:./bin"
  ```


### Wallet setting
Let's define additional shell variables. `KEY_NAME` value of your choice, I uses a value of “crypto”:
  ```bash
  $ KEY_NAME="crypto"
  $ KEYRING_BACKEND="test"
  ```
Derive a new key locally:
  ```bash
  $ akash --keyring-backend "$KEYRING_BACKEND" keys add "$KEY_NAME"
  ```

You'll see a response similar to below:
  ```yaml
  - name: crypto
    type: local
    address: akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5
    pubkey: akashpub1addwnpepqvc7x8r2dyula25ucxtawrt39henydttzddvrw6xz5gvcee4arfp7ppe6md
    mnemonic: ""
    threshold: 0
    pubkeys: []


  **Important** write this mnemonic phrase in a safe place.
  It is the only way to recover your account if you ever forget your password.

  share mammal walnut direct plug drive cruise real pencil random regret chunk use live entire gloom donate require autumn bid brown impact scrap slab
  ```
In the above example, your new Akash address is `akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5`, define a new shell variable with it:
  ```bash
  $ ACCOUNT_ADDRESS="akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5"
  ```

### Funding your account
In this guide I use the edgenet Akash network. Non-mainnet networks will often times have a "faucet" running - a server that will send tokens to your account. You can see the faucet url by running:
  ```bash
  $ curl "$AKASH_NET/faucet-url.txt"

  https://akash.vitwit.com/faucet
  ```
Go to the resulting URL and enter your account address; you should see tokens in your account shortly.
Check your account balance with:
  ```yaml
  $ akash --node "$AKASH_NODE" query bank balances "$ACCOUNT_ADDRESS"

  balances: 
  - amount: "100000000" 
   denom: uakt 
  pagination: 
   next_key: null 
   total: "0"
  ```
As you can see, the balance is non-zero and contains 100000000 uakt.
Okay, now you're ready to deploy the application.



# Deploying MySQL and Adminer


Make sure you have the following set of variables defined on your shell in the previous step "Installing and preparing Akash":
  * `AKASH_CHAIN_ID` - Chain ID of the Akash network connecting to
  * `AKASH_NODE` - Akash network configuration base URL
  * `KEY_NAME` - The name of the key you will be deploying from
  * `KEYRING_BACKEND` - Keyring backend to use for local keys
  * `ACCOUNT_ADDRESS` - The address of your account

### Creating a deployment configuration
For configuration in Akash uses [Stack Definition Language (SDL)](https://docs.akash.network/v/master/documentation/sdl) similar to Docker Compose files. Deployment services, datacenters, pricing, etc.. are described by a YAML configuration file. These files may end in .yml or .yaml.
Create `deploy.yml` configuration file:
  ```bash
  $ nano deploy.yml
  ```
With following content:  

```yaml
version: "2.0"

services:
  db:
    image: mysql
    env:
        - MYSQL_ROOT_PASSWORD=12345
    expose:
      - port: 3306
        to:
          - global: true
  
  adminer:
    image: adminer
    depends-on:
        - db
    expose:
      - port: 8080
        as: 80
        to:
          - global: true

profiles:
  compute:
    db:
      resources:
        cpu:
          units: 0.5
        memory:
          size: 512Mi
        storage:
          size: 512Mi
    
    adminer:
      resources:
        cpu:
          units: 0.5
        memory:
          size: 512Mi
        storage:
          size: 512Mi
          
  placement:
    westcoast:    
      pricing:
        db: 
          denom: uakt
          amount: 5000
        adminer: 
          denom: uakt
          amount: 5000

deployment:
  db:
    westcoast:
      profile: db
      count: 1
  adminer:
    westcoast:
      profile: adminer
      count: 1
```

Here we define two services: `db` for the database instance and `adminer` for the Adminer utility. Official `mysql` and `adminer` images from Docker Hub are used as Docker images. For each service specified its deployment parameters. The `services.web.images` field is the name of needed Docker image. The `services.expose.port` field means the container port to expose. The `services.expose.as` field is the port number to expose the container port as specified.  

For the `db` service, define the `MYSQL_ROOT_PASSWORD` environment variable , which specifies the password for the `root` user to login.
MySQL runs on port `3306`, so expose it. For the `adminer` service, specify the dependency on the` db` service using `depends-on` parameter and redirect access to the service to port` 80`.

Next section `profiles.compute` is map of named compute profiles. Each profile specifies compute resources to be leased for each service instance uses uses the profile. Both defined profiles having resource requirements of 0.5 vCPUs, 512 megabytes of RAM memory, and 512 megabytes of storage space available. `cpu.units` represent a virtual CPU share and can be fractional.

`profiles.placement` is map of named datacenter profiles. Each profile specifies required datacenter attributes and pricing configuration for each compute profile that will be used within the datacenter. This defines a profile named `westcoast` having required attribute `pricing` with a max price for the `db` and `adminer` compute profiles of 5000 uakt per block. 

The `deployment` section defines how to deploy the services. It is a mapping of service name to deployment configuration.
Each service to be deployed has an entry in the deployment. This entry is maps datacenter profiles to compute profiles to create a final desired configuration for the resources required for the service.


### Creating the deployment
In this step, you post your deployment, the Akash marketplace matches you with a provider via auction. To deploy on Akash, run:
  ```bash
  $ akash tx deployment create deploy.yml --from $KEY_NAME \
    --node $AKASH_NODE --chain-id $AKASH_CHAIN_ID \
    --keyring-backend $KEYRING_BACKEND -y
  ```
Make sure there are no errors in the command response. The error information will be in the `raw_log` responce field.


### Wait for your lease
You can check the status of your lease by running:
  ```
  $ akash query market lease list --owner $ACCOUNT_ADDRESS \
    --node $AKASH_NODE --state active
  ```
  You should see a response similar to:
```yaml
leases:
- lease_id:
    dseq: "204226"
    gseq: 1
    oseq: 1
    owner: akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5
    provider: akash15ql9ycjkkxhpc2nxtnf78qqjguwzz8gc4ue7wl
  price:
    amount: "1111"
    denom: uakt
  state: active
pagination:
  next_key: null
  total: "0"
```

From this response we can extract some new required for future referencing shell variables:
  ```bash
  $ PROVIDER="akash15ql9ycjkkxhpc2nxtnf78qqjguwzz8gc4ue7wl"
  $ DSEQ="204226"
  $ OSEQ="1"
  $ GSEQ="1"
  ```


### Uploading manifest
Upload the manifest using the values from above step:
  ```bash
  $ akash provider send-manifest deploy.yml \
    --node $AKASH_NODE --dseq $DSEQ \
    --oseq $OSEQ --gseq $GSEQ \
    --owner $ACCOUNT_ADDRESS --provider $PROVIDER
  ```
Your image is deployed, once you uploaded the manifest. You can retrieve the access details by running the `akash provider lease-status` command:
  ```bash
  $ akash provider lease-status \
    --node $AKASH_NODE --dseq $DSEQ \
    --oseq $OSEQ --gseq $GSEQ \
    --provider $PROVIDER --owner $ACCOUNT_ADDRESS
  ```
You should see a response similar to:
```json
  {
    "services": {
      "adminer": {
        "name": "adminer",
        "available": 1,
        "total": 1,
        "uris": [
          "54ts8qmuk5flb4aq3sektgcrsg.provider4.akashdev.net"
        ],
        "observed-generation": 0,
        "replicas": 0,
        "updated-replicas": 0,
        "ready-replicas": 0,
        "available-replicas": 0
      },
      "db": {
        "name": "db",
        "available": 1,
        "total": 1,
        "uris": null,
        "observed-generation": 0,
        "replicas": 0,
        "updated-replicas": 0,
        "ready-replicas": 0,
        "available-replicas": 0
      }
    },
    "forwarded-ports": {
      "db": [
        {
          "host": "ext.provider4.akashdev.net",
          "port": 3306,
          "externalPort": 30661,
          "proto": "TCP",
          "available": 1,
          "name": "db"
        }
      ]
    }
  }
  ```

### Access to the database
##### Console
Let's connect to our remote MySQL server in console mode. To do this, you must have the `mysql-client` package installed.  
First, we need to find out the connection data, such as host and port. The given SDL exposes your database service on a random port. 
You can find out the generated hostname and port for your deployment with the `akash provider lease-status` command as described above.
The response will contain a list of `"forwarded ports"`:
  ```json
  "forwarded-ports": {
      "db": [
        {
          "host": "ext.provider4.akashdev.net",
          "port": 3306,
          "externalPort": 30661,
          "proto": "TCP",
          "available": 1,
          "name": "db"
        }
      ]
    }
  ```
Look for an entry for port `3306`; the `host` and `externalPort` fields from that are where you can reach your remote database. For me host is `ext.provider4.akashdev.net` and post is `30661`.  

To log into the console of the remote MySQL server use the following command, replace `<host>` and `<port>` with the obtained above data:
```bash
$ mysql -u root -p -h <host> -P <port>
```
Then you will need to enter the password for the root user that you specified in the deployment configuration file and you will be taken to the command line of the remote mysql server.

Let's create a simple database with one table and some data in it for testing:
```sql
CREATE DATABASE test;
USE test

CREATE TABLE `table1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `key_name` varchar(20) NOT NULL,
  `value` varchar(50) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO `table1` (`id`, `key_name`, `value`) VALUES 
(1, 'key 1', 'value 1'), (2, 'key 2', 'value 2');
```

Now let's query the table to make sure everything was created successfully:
```sql
mysql> SELECT * FROM table1;

+----+----------+---------+
| id | key_name | value   |
+----+----------+---------+
|  1 | key 1    | value 1 |
|  2 | key 2    | value 2 |
+----+----------+---------+
2 rows in set (0.24 sec)
```

##### Adminer
Now let's check our database through the web interface using the Adminer utility. The link can be found from the same output of the `akash provider lease-status` command. In the `services` list, find the `uri` for the `adminer` service:
  ```json
  "services": {
      "adminer": {
        "name": "adminer",
        "available": 1,
        "total": 1,
        "uris": [
          "54ts8qmuk5flb4aq3sektgcrsg.provider4.akashdev.net"
        ],
        "observed-generation": 0,
        "replicas": 0,
        "updated-replicas": 0,
        "ready-replicas": 0,
        "available-replicas": 0
       },
  ```
In my case the link is http://54ts8qmuk5flb4aq3sektgcrsg.provider4.akashdev.net. We follow the found link and get into the Adminer interface.
Now you need to enter your login information: select `MySQL` in the `System` dropdown, in the `Server` field you need to enter the host and port in the format `host:port` (in my case it looks like `ext.provider4.akashdev.net:30661`). Next, enter Username and password and enter.

![adminer1](https://github.com/yuravorobei/mysql-on-akash-guide/blob/main/media/adminer_1.png)

After logging in, we will see a list of all databases on the server and make sure that the list contains the previously created `test` database. Also make sure you have the table and data in the database you created earlier:
![adminer2](https://github.com/yuravorobei/mysql-on-akash-guide/blob/main/media/adminer_2.png)


### Service Logs
You can view your application logs to debug issues or watch progress using `akash provider service-logs` command, for example:
```
  $ akash provider service-logs --node "$AKASH_NODE" --owner "$ACCOUNT_ADDRESS" \
  --dseq "$DSEQ" --gseq 1 --oseq $OSEQ --provider "$PROVIDER" \
  --service web \
```

You should see a response similar to this:
```
[adminer-7c764fc58b-zhh2t] [Fri Dec 11 16:18:27 2020] PHP 7.4.13 Development Server (http://[::]:8080) started
[adminer-7c764fc58b-zhh2t] [Fri Dec 11 16:18:43 2020] [::ffff:10.233.90.1]:55204 Accepted
[adminer-7c764fc58b-zhh2t] [Fri Dec 11 16:18:44 2020] [::ffff:10.233.90.1]:55204 [200]: GET /
[adminer-7c764fc58b-zhh2t] [Fri Dec 11 16:18:44 2020] [::ffff:10.233.90.1]:55204 Closing
[adminer-7c764fc58b-zhh2t] [Fri Dec 11 16:18:44 2020] [::ffff:10.233.90.1]:55316 Accepted
[adminer-7c764fc58b-zhh2t] [Fri Dec 11 16:18:44 2020] [::ffff:10.233.90.1]:55316 [200]: GET /?file=default.css&version=4.7.8

.........

[db-577569889f-b2464] 2020-12-11 16:18:27+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.22-1debian10 started.
[db-577569889f-b2464] 2020-12-11 16:18:27+00:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
[db-577569889f-b2464] 2020-12-11 16:18:27+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.22-1debian10 started.
[db-577569889f-b2464] 2020-12-11 16:18:27+00:00 [Note] [Entrypoint]: Initializing database files
[db-577569889f-b2464] 2020-12-11T16:18:27.909611Z 0 [System] [MY-013169] [Server] /usr/sbin/mysqld (mysqld 8.0.22) initializing of server in progress as process 43
[db-577569889f-b2464] 2020-12-11T16:18:27.914369Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
```

### Close your deployment

When you are done with your application, close the deployment. This will deprovision your container and stop the token transfer. Close deployment using deployment by creating a deployment-close transaction:
  ```shell
  $ akash tx deployment close --node $AKASH_NODE \
    --chain-id $AKASH_CHAIN_ID --dseq $DSEQ \
    --owner $ACCOUNT_ADDRESS --from $KEY_NAME \
    --keyring-backend $KEYRING_BACKEND -y
  ```

Additionally, you can also query the market to check if your lease is closed:
  ```bash
  $ akash query market lease list --owner $ACCOUNT_ADDRESS --node $AKASH_NODE
  ```
You should see a response similar to:
  ```yaml
  - lease_id:
      dseq: "204226"
      gseq: 1
      oseq: 1
      owner: akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5
      provider: akash15ql9ycjkkxhpc2nxtnf78qqjguwzz8gc4ue7wl
    price:
      amount: "1111"
      denom: uakt
    state: closed
  pagination:
    next_key: null
    total: "0"
  ```
As you can notice from the above, you lease will be marked `closed`.
