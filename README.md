# ztncui - ZeroTier network controller user interface

ztncui is a web user interface for a standalone [ZeroTier](https://zerotier.com) network controller.

Screenshots can be seen at [key-networks.com/ztncui](https://key-networks.com/ztncui).

Follow us on [![alt @key_networks on Twitter](https://i.imgur.com/wWzX9uB.png)](https://twitter.com/key_networks)

## Packages
Instructions for installing on Linux from RPM or DEB packges are available at [key-networks.com/ztncui](https://key-networks.com/ztncui).

## Docker Container Image
See [https://github.com/key-networks/ztncui-aio](https://github.com/key-networks/ztncui-aio)

# Local Installation Guide
> [!WARNING]
> ztncui has been developed on a Linux platform and expects the ZT home directory to be in `/var/lib/zerotier-one`.

> [!NOTE]
> Paths are relative to the cloned ztncui directory.

## Prerequisites
**ZeroTier One**<br>
[ZeroTier One](https://www.zerotier.com/download/) must be installed on the same machine. This will run as the network controller to establish ZeroTier networks.

**Node.js**<br>
ztncui is a [Node.js](https://nodejs.org) [Express](https://expressjs.com) application. It requires v14.

**argon2**<br>
Used for password hashing. Argon needs g++ and node-gyp.
```
sudo npm install -g node-gyp
```

## Installing
### Step 1: Clone the Repository
Clone the repo on a machine running ZeroTier One.
```shell
git clone https://github.com/key-networks/ztncui
```

### Step 2: Install Node.js Packages
```shell
cd ztncui/src &&
npm install
```

## Configuring
### Step 3: Make .env File
In the root of the ztncui directory, create a `.env` file with the content:
```
ZT_TOKEN=
NODE_ENV=production
```
**ZT_TOKEN**<br>
The app needs to know the zerotier-one authtoken.secret which can be found at `/var/lib/zerotier-one/authtoken.secret`. Paste this token after `ZT_TOKEN=` in the `.env` file.

**MODE_ENV**<br>
To run the server in production mode, set this to `production`. While not required, the template engine always re-compiles the pug file when rendering without this (taking ~200 ms!).

#### Optional Environment Variables
**ZT_ADDR**<br>
You can specify a different address for the zerotier-one API (which defaults to localhost:9993):
```
ZT_ADDR=localhost:9995
```

**HTTP_ALL_INTERFACES**<br>
The app can be made to listen on all interfaces for HTTP requests by setting `HTTP_ALL_INTERFACES`, e.g.:
```
HTTP_ALL_INTERFACES=yes
```
Note that HTTP traffic is unencrypted, so this should only be done on a secure network, otherwise usernames and passwords will be exposed in plain text over the network.

**HTTP_PORT**<br>
The app runs on TCP port 3000 by default.  If port 3000 is already in use, you can specify a different port, e.g.:
```
HTTP_PORT=3456
```

**HTTPS_PORT & HTTPS_HOST**<br>
The app can be made to listen on all interfaces for HTTPS requests by specifying `HTTPS_PORT`, e.g.:
```
HTTPS_PORT=3443
```
The app can be made to listen on a specific interface for HTTPS requests by specifying `HTTPS_HOST` (the host name or IP address of the interface) along with `HTTPS_PORT`, e.g.:
```
HTTPS_PORT=3443
HTTPS_HOST=12.34.56.78
```
#### Set Permissions
After all edits to the `.env` file have been made, make the `.env` readable by the user running ztncui only:
```shell
chmod 400 .env
chown ztncui:ztncui .env
```

### Step 4: Copy Default Passwd File
> [!CAUTION]
> Do not confuse `etc/passwd` with `/etc/passwd`!

To prevent git from overwriting your password file every time you pull updates from the repository, the `etc/passwd` file has been added to `.gitignore`.  The default file will need to be copied after the first time you do a git clone.
```shell
cp -v etc/default.passwd etc/passwd
```

### Step 5: Start the App
#### Manually
```shell
npm start
```
#### Automatically (with [PM2](http://pm2.keymetrics.io))
To start the app automatically, something like PM2 can be used. Install it with:
```shell
sudo npm install -g pm2
```

Add ztncui as a managed app with:
```shell
pm2 start bin/www --name ztncui
```

To detect the init system:
```shell
pm2 startup
```

PM2 will then give you a command to execute to configure automatic startup of PM2 for your system.


Save the current PM2 process list so that ztncui will restart across reboots:
```shell
pm2 save
```
### Step 6: Access the App
#### HTTP
This app listens for HTTP requests on the looback interface (default port 3000). If the host machine has a GUI and GUI web browser then use it to access the app. The default URL will be [http://localhost:3000](http://localhost:3000). Alternatively, use a text web browser like Lynx or a CLI web browser like curl:
```shell
curl http://localhost:3000
```
You should see the front page of the app (or the raw HTML with curl).
#### HTTPS
The app can be reverse proxied by Nginx (which can proxy the HTTP as HTTPS), or accessed over an SSH tunnel as described below.
##### TLS Certificate
For HTTPS you will need a TLS (SSL) certificate and private key pair. There are a few options:
1. By default, if there is no existing TLS certificate and private key pair, the RPM and DEB packages automatically generate a self-signed certificate / private key pair. If you are running directly from source, then generate a self-signed certificate as follows:
   ```shell
   cd etc/tls
   openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout privkey.pem -out fullchain.pem
   ```
   Fill in the required details as prompted.

   The advantage of this option is that it is quick and easy to generate the certificate / private key pair.  The disadvantage is that your web browser will give you a warning that it cannot verify the certificate.  You can override this warning and make a temporary exception.
2. Buy a certificate and store the private key as `etc/tls/privkey.pem` and the full certificate chain as `etc/tls/fullchain.pem`.  They need to be in PEM format.
3. Get a free certificate from Letsencrypt.org:

   a. Install certbot by following the instructions at certbot.eff.org:

      i.   For "Software" select "None of the above".
      ii.  For "System" select your OS.
      iii. Follow the instructions to install certbot on your system.

    b. Use certbot to generate a certificate in webroot mode from the root of the ztncui directory:
      ```shell
      certbot --webroot -w public -d [network_controller_fqdn]
      ```
      Where **[network_controller_fqdn]** is the FQDN that resolves back to the address of the machine running the ZeroTier network controller and ztncui.

      If certbot runs successfully, it should give you the location of your certificate, which should be something like:
      ```
      /etc/letsencrypt/live/[network_controller_fqdn]/fullchain.pem
      ```

      c. Make soft links from etc/tls to the certificate and private key under /etc/letsencrypt/live:
      ```shell
      cd etc/tls
      ln -s /etc/letsencrypt/live/[network_controller_fqdn]/fullchain.pem
      ln -s /etc/letsencrypt/live/[network_controller_fqdn]/privkey.pem
      ```

      d. Take note of the options for renewing Letsencrypt certificates and implement an appropriate strategy.

###### Test HTTPS access
Once you have a certificate at `etc/tls/fullchain.pem` and private key at `etc/tls/privkey.pem`, you should be able to access ztncui over HTTPS on the port specified by `HTTPS_PORT`.

##### Remote access via SSH
###### SSH tunnel from Linux / Unix / macOS client
An SSH tunnel can be established with:
```shell
ssh -f user@network.controller.machine -L 3333:localhost:3000 -N
```
where:
* **network.controller.machine** is the FQDN of the machine running the ZT network controller and ztncui, and
* **user** is any user account that you have on that machine.

Once the SSH tunnel has been established, access the ztncui web interface in a web browser on your local machine at: http://localhost:3333

###### SSH tunnel from a Windows machine
On Windows you can install [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)

Open PuTTY and configure as follows:

1. Go to Connection -> SSH -> Tunnels.
2. Set **Source port** to 3333
3. Set **Destination** to localhost:3000
4. Click on the **Add** button.
5. Go to **Session** in the **Category** panel on the left.
6. Set **Host Name (or IP address)** to the FQDN of the machine running the ZT network controller and ztncui.
7. Enter a name for the configuration in **Saved Sessions** and click **Save**.
8. Click the **Open** button and log into the network controller machine.

Once the SSH tunnel has been established, access the ztncui web interface in a web browser on your local machine at: http://localhost:3333

# Usage
## User Accounts
Once you have access to the web UI of ztncui, log in as user **admin** with password **password**.

You will be prompted to change the default password.

It's a good idea to create your own username and delete the default admin account.  You can do this by clicking on the **Users** tab and then the **Create user** tab.  Note that you then have to log out and log in as the new user before you can delete the default admin account.

## Networks
Click on the **Home** tab to get to the network controller home page.  From there you can click on the **Networks** tab to see the existing networks configured on the network controller (probably none if you have just set it up).

### Create a New Network
Click on the **Add network** tab to create a new ZeroTier network that is controlled by the network controller.  Give it a name and click **Create Network**.  You will then be taken back to the **Networks** page that lists all the networks on the controller.

#### Easy Network Setup
On the **Networks** page, click **easy setup** for the network that you want to auto-configure.  Click **Generate network address** to assign a random network address, or manually enter the network address in CIDR notation.  The start and end of the IP assignment pool will be automatically calculated, but these can be manually adjusted.  Click **Submit** to apply the configuration.  You should then get a notice that the network setup succeeded.

Note that the **easy setup** only works for IPv4 at this stage.  To set up IPv6, follow the **detail** link for a network from the **Networks** page and set up each property manually.

### Manage a Network
#### Join Devices to a Network
Invite users to join the network with:
```shell
sudo zerotier-cli join ################
```
where ################ is the 16-digit ZeroTier network ID.

Get the user to send you their 10-digit ZeroTier address, which they can get by running:
```shell
sudo zerotier-cli status
```
#### Authorize Members on a Network
On the **Networks** page, click **members** to see the devices which are trying to join the network.  Use the ZeroTier address given to you by the user to identify them and name them appropriately under **Member name**.

Then check the **Authorized** checkbox to authorize the user on the network.

If the user's device is online and you click the **Refresh** button, you should see their IP assignment being populated.

Once two or more members are authorized on the network, they should be able to connect to each other via their assigned IP addresses.

#### IP Assignments
IP assignments can be changed by clicking on the IP address in the **members** page.  Enter an IP address in the managed route subnet and click the **+** icon.  Then delete the old IP address.

#### Ethernet bridging
Ethernet bridging between virtual and physical networks can be enabled by checking the **Active bridge** checkbox on the **members** page.

#### Network detail
On the **Networks** page, click **detail** to see the detail of a network.

Note that certain properties can be set by clicking on them - e.g.:
* ipAssignmentPools
* name
* routes
* v4AssignMode
* v6AssignMode

Note that editing of certain properties, such as rules and tags, has not been implemented yet.  Please feed back on your requirements.

#### Member detail
On the network **detail** page and on the **members** page, if you click on the member ID, you will end up on the member detail page.


### Delete a Network
On the **Networks** page, click the trash can icon to delete a network.  You will be warned that this action cannot be undone.  Click the **Delete** button to confirm the action.

### Change Network Name
On the **Networks** page, click the name of the network to rename it.

# Reference

### Summary of listening states
| Environment variable | Protocol | Listen On      | Port              |
| :------------------: | :------: | :-------:      | :--:              |
| [none]               | HTTP     | localhost      | 3000              |
| HTTP_PORT            | HTTP     | localhost      | HTTP_PORT         |
| HTTP_ALL_INTERFACES  | HTTP     | all interfaces | HTTP_PORT || 3000 |
| HTTPS_PORT           | HTTPS    | all interfaces | HTTPS_PORT        |
| HTTPS_HOST           | HTTPS    | HTTPS_HOST     | HTTPS_PORT        |

# Feedback
Please give us your feedback... good, bad or ugly.  Constructive criticism is welcomed.  Please use the contact form at [key-networks.com](https://key-networks.com/) - Thanks :)

## Bug and Vulnerability Reporting
Problems with ztncui can be reported using the GitHub issue tracking system.  Please use the contact form at [key-networks.com](https://key-networks.com/) to privately report potential vulnerabilities.  Thank you.

## License
The ztncui code is open source code, licensed under the GNU GPLv3, and is free to use on those terms. If you are interested in commercial licensing, please contact us via the contact form at [key-networks.com](https://key-networks.com) .

## Thanks
- @lideming for a rework and improvement of the network details page, adding DNS support, peer status/address/latency and other improvements.
- @Koromix for a fix for incompatibility with ZeroTier 1.12.
