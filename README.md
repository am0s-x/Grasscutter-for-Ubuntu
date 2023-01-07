# Grasscutter on Ubuntu server 20.04

### Install MongoDB-org

Import pgp key
```bash
wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -
```

Create listfile
```bash
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
sudo apt update
```

Install latest MongoDB
```bash
sudo apt install -y mongodb-org
```

Start `mongod` service
```bash
sudo systemctl enable mongod
sudo systemctl start mongod
sudo systemctl status mongod
# in case of `systemctl` error, please make sure the CPU type is set to `host` if using Proxmox
```

> Ref: https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/



### Install Java SDK

Download and extract Java Development Kit
```bash
wget https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.tar.gz 
```

Extract JDK
```bash
sudo tar -zxvf jdk-17_linux-x64_bin.tar.gz -C /usr/local
```

Install JDK
```bash
sudo mv /usr/local/jdk-17.0.5 /usr/local/java
```

Configure environment variables
```bash
sudo nano /etc/profile

# paste the following to the end
export JAVA_HOME=/usr/local/java
export PATH=$PATH:$JAVA_HOME/bin;
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar;

# update environment variables
source /etc/profile

# check java version to verify installation
java -version
```

Verify Python3
``` bash
# Grasscutter requires at least Python 3.8 which should comes with Ubuntu 20.04
python3 --version
```



### Install Grasscutter

Clone repo
```bash
git clone https://github.com/Grasscutters/Grasscutter.git
```

Compile the server
```bash
cd Grasscutter
chmod +x gradlew
./gradlew jar
```

Or download server from release
```bash
wget https://github.com/Grasscutters/Grasscutter/releases/download/v1.4.3/grasscutter-1.4.3.jar
```

Download and extract resources
> https://git.crepe.moe/grasscutters/Grasscutter_Resources
```bash
cd ~
wget https://git.crepe.moe/grasscutters/Grasscutter_Resources/-/raw/main/Grasscutter_Resources-3.3.zip
sudo apt install unzip
unzip Grasscutter_Resources-3.3.zip
```

Move resources to Grasscutter directory
```bash
mv Grasscutter_Resources-3.3/Resources/ ~/Grasscutter/
```



### Install `mitmproxy` proxy 

Download and install
```bash
wget https://snapshots.mitmproxy.org/8.1.0/mitmproxy-8.1.0-linux.tar.gz

# unzip and move it to `/usr/bin`
sudo tar -zxvf mitmproxy-8.1.0-linux.tar.gz -C /usr/bin

# verify installation
mitmproxy --version && mitmdump --version && mitmweb --version
# this should return the same version info for all three program
```

Start and test proxy
```bash
# start proxy with `proxy.py` config file
cd ~/Grasscutter/
mitmdump -s proxy.py --ssl-insecure --set block_global=false --listen-port 54321
# or without specify port #, using default port `8080`
mitmdump -s proxy.py --ssl-insecure --set block_global=false
# `--set block_global=false` enables connection from other machine, set to `true` allow only localhost

# test proxy
curl -x "127.0.0.1:54321" "http://mitm.it/"
# successful if returns "...Grasscutter server..."

# stop the proxy upon successful test results
CTRL + C
```



### Configure the Grasscutter server

Edit the `proxy_config.py` file
```bash
nano ~/Grasscutter/proxy_config.py
```

Point the `REMOTE_HOST` to the server's IP addr:
```python
import os

# This can also be replaced with another IP address.
USE_SSL = True
REMOTE_HOST = "192.168.x.x"
REMOTE_PORT = 443

if os.getenv('MITM_REMOTE_HOST') != None:
    REMOTE_HOST = os.getenv('MITM_REMOTE_HOST')
if os.getenv('MITM_REMOTE_PORT') != None:
    REMOTE_PORT = int(os.getenv('MITM_REMOTE_PORT'))
if os.getenv('MITM_USE_SSL') != None:
    USE_SSL = bool(os.getenv('MITM_USE_SSL'))

print('MITM Remote Host: ' + REMOTE_HOST)
print('MITM Remote Port: ' + str(REMOTE_PORT))
print('MITM Use SSL ' + str(USE_SSL))
```

Edit `_server.http.accessAddress_` and `_server.game.accessAddress_` from `127.0.0.1` to the real `LAN IP`, e.g. `192.168.x.x`
```bash
nano ~/Grasscutter/config.json
```
Example:
```json
...
  "server": {
    "debugWhitelist": [],
    "debugBlacklist": [],
    "runMode": "HYBRID",
    "logCommands": false,
    "http": {
      "bindAddress": "0.0.0.0",
      "bindPort": 443,
      "accessAddress": "192.168.x.x",
      "accessPort": 0,
      "encryption": {
        "useEncryption": true,
        "useInRouting": true,
        "keystore": "./keystore.p12",
        "keystorePassword": "123456"
      },
...
    "game": {
      "bindAddress": "0.0.0.0",
      "bindPort": 22102,
      "accessAddress": "192.168.x.x",
      "accessPort": 0,
...
```



### Install cert

Download cert from proxy
```bash
# .pem for Linux/iOS, .p12 for Win
wget -e "http_proxy=127.0.0.1:54321" http://mitm.it/cert/pem -O mitmproxy-ca-cert.pem
wget -e "http_proxy=127.0.0.1:54321" http://mitm.it/cert/p12 -O mitmproxy-ca-cert.p12
```

Or find cert from `~/.mitmprox/` directly
```bash
cd ~/.mitmproxy/
```

Convert from `.pem` to `.crt`
```bash
# convert .pem to .crt
openssl x509 -in mitmproxy-ca-cert.pem -inform PEM -out mitmproxy-ca-cert.crt
# OR convert .cer to .crt
openssl x509 -in mitmproxy-ca-cert.cer -inform DRE -out mitmproxy-ca-cert.crt
```
> Ref: 
> - https://askubuntu.com/questions/73287/how-do-i-install-a-root-certificate/94861#94861
> - https://docs.mitmproxy.org/stable/concepts-certificates/

Install the new cert
```bash
# make extra directory for the new cert
sudo mkdir /usr/local/share/ca-certificates/extra

# copy the new cert
sudo cp ~/.mitmproxy/mitmproxy-ca-cert.crt /usr/local/share/ca-certificates/extra/
sudo cp ~/.mitmproxy/mitmproxy-ca-cert.crt /usr/share/ca-certificates/

# install the new cert
sudo apt install ca-certificates -y
sudo dpkg-reconfigure ca-certificates
```
> Endpoint will also need to download and install the certs.
> Connect the endpoints to proxy server, then open `http://mitm.it/` to donwload certs.



### Configure `hosts` file on the server
```bash
# open `hosts` on the server
sudo nano /etc/hosts

# add the following lines and save

127.0.0.1 dispatchosglobal.yuanshen.com
127.0.0.1 dispatchcnglobal.yuanshen.com
127.0.0.1 osusadispatch.yuanshen.com
127.0.0.1 oseurodispatch.yuanshen.com
127.0.0.1 osasiadispatch.yuanshen.com

127.0.0.1 hk4e-api-os-static.mihoyo.com
127.0.0.1 hk4e-api-static.mihoyo.com
127.0.0.1 hk4e-api-os.mihoyo.com
127.0.0.1 hk4e-api.mihoyo.com
127.0.0.1 hk4e-sdk-os.mihoyo.com
127.0.0.1 hk4e-sdk.mihoyo.com

127.0.0.1 account.mihoyo.com
127.0.0.1 api-os-takumi.mihoyo.com
127.0.0.1 api-takumi.mihoyo.com
127.0.0.1 sdk-os-static.mihoyo.com
127.0.0.1 sdk-static.mihoyo.com
127.0.0.1 webstatic-sea.mihoyo.com
127.0.0.1 webstatic.mihoyo.com
127.0.0.1 uploadstatic-sea.mihoyo.com
127.0.0.1 uploadstatic.mihoyo.com

127.0.0.1 api-os-takumi.hoyoverse.com
127.0.0.1 sdk-os-static.hoyoverse.com
127.0.0.1 sdk-os.hoyoverse.com
127.0.0.1 webstatic-sea.hoyoverse.com
127.0.0.1 uploadstatic-sea.hoyoverse.com
127.0.0.1 api-takumi.hoyoverse.com
127.0.0.1 sdk-static.hoyoverse.com
127.0.0.1 sdk.hoyoverse.com
127.0.0.1 webstatic.hoyoverse.com
127.0.0.1 uploadstatic.hoyoverse.com
127.0.0.1 account.hoyoverse.com
127.0.0.1 api-account-os.hoyoverse.com
127.0.0.1 api-account.hoyoverse.com

127.0.0.1 hk4e-api-os.hoyoverse.com
127.0.0.1 hk4e-api-os-static.hoyoverse.com
127.0.0.1 hk4e-sdk-os.hoyoverse.com
127.0.0.1 hk4e-sdk-os-static.hoyoverse.com
127.0.0.1 hk4e-api.hoyoverse.com
127.0.0.1 hk4e-api-static.hoyoverse.com
127.0.0.1 hk4e-sdk.hoyoverse.com
127.0.0.1 hk4e-sdk-static.hoyoverse.com

0.0.0.0 log-upload.mihoyo.com
0.0.0.0 log-upload-os.mihoyo.com
0.0.0.0 log-upload-os.hoyoverse.com
0.0.0.0 devlog-upload.mihoyo.com
0.0.0.0 overseauspider.yuanshen.com
```



### Run Grasscutter server

> run the following server commands from the `Glasscutter` directory
```bash
cd ~/Grasscutter/

# start proxy, default port `8080`
mitmdump -s proxy.py --ssl-insecure --set block_global=false

# open another SSH session to run the server
# requires `sudo` here as port `443` is reserved even if it is not in use
sudo /usr/local/java/bin/java -jar grasscutter-<version>.jar -handbook
```
> Grasscutter server commands can be found at
> https://github.com/Grasscutters/Grasscutter/wiki/Commands
> https://wmn1525.github.io/grasscutterTools/dist/index.html#/
> https://m.clinicmed.net/article/17206.html
> 
> GM handbook can be found at
> `~/Grasscutter/GM Handbook/`

E.g. to create user accounts
```java
account create <username> <uid>
```

To stop the server
```java
stop
```
> Do **NOT** stop the server process by `CTRL + C`



### Connect to the server

1. on the endpoint, enable proxuy and point to proxy server at its IP addr, e.g. `192.168.x.x`, and the proxy port #, e.g. `8080`
2. visit `http://mitm.it/` to download and install the cert to the trusted root cert directory
3. On Windows, apply the RSApatch at https://github.com/34736384/RSAPatch, for v3.3.0
4. Download and install Cultivation at https://github.com/Grasscutters/Cultivation, and set the proxy address to the IP addr above, and enable HTTPS.
5. Launch the game using Cultivation. If it shows Hoyoverse, its running the private server, if its Mihoyo, then its connected to official server.



---



### Miscellaneous 

##### References
- Grasscutter repo - https://github.com/Grasscutters/Grasscutter
- Grasscutter Discord - https://discord.gg/T5vZU6UyeG
- RSApatch for Win - https://github.com/34736384/RSAPatch
- https://iamxiaokong.cn/170/
- https://blog.tomys.top/2022-04/GenshinTJ/
- Install MongoDB - https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/
- Install cert for Ubuntu - https://ubuntu.com/server/docs/security-trust-store


##### MacOS test open port
```bash
nc -vnzu <ip> <port|port-range>
```


### Basic topology

`Endpoints` -> `proxy-server:8080` -> `grasscutter-server:443`

> `proxy-server` and `grasscutter-server` could be on the same server.
> 
> `endpoints` and `proxy-server` and `grasscutter-server` can also on one machine.
