# Manual compilation and start of TON validator on RHEL-based Linux (CentOS/Fedora/RHEL)

#### Installation of dependings packages:

```bash
sudo yum install -y epel-release
sudo dnf config-manager --set-enabled PowerTools
sudo yum install -y git make cmake clang gflags gflags-devel zlib zlib-devel openssl-devel openssl-libs readline-devel libmicrohttpd python3 python3-pip python36-devel
```

#### Compilation of TON full node:

```bash
git clone https://github.com/ton-blockchain/ton.git
cd ton
git submodule update --init --recursive
cd ../
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ../ton
cmake --build .
```

Than main parts of TON blockchain full node you can find here:

`build/validator-engine/validator-engine`

`build/lite-client/lite-client`

`build/crypto/fift`

For more useful environment you can add into `.bashrc` these lines (all directories only as example):

```bash
alias lite-client="~/apps/fullnode-build/lite-client/lite-client -a 134.249.174.87:7654 -p /home/tgnetwork/ton-fullnode/ton-validation/liteserver.pub"
alias func='~/apps/fullnode-build/crypto/func'
alias fift='~/apps/fullnode-build/crypto/fift -I ~/apps/ton/crypto/fift/lib'
alias validator-engine='~/apps/fullnode-build/validator-engine/validator-engine'
```

#### Setting up a validator database:

```bash
sudo mkdir /var/ton-work
cd /var/ton-work
mkdir etc
cd etc
wget https://newton-blockchain.github.io/newton-test.global.config.json
cd ../
mkdir db
```

First launch of `validator-engine` for creating of a `db/config.json`:

```bash
validator-engine -C /var/ton-work/etc/newton-test.global.config.json --db /var/ton-work/db/ --ip <IP>:<PORT> -l /var/ton-work/log
```

After that you need to create `server/server.pub`, `client/client.pub`, `liteserver/liteserver.pub` keypairs, as it was described at https://ton.org/FullNode-HOWTO.txt

After creation of keypairs and editing db/config.json you can start a fully working full node:

```bash
sudo /home/tgnetwork/apps/fullnode-build/validator-engine/validator-engine -d --db /var/ton-work/db/ -C /var/ton-work/etc/newton-test.global.config.json  -l /var/ton-work/log -S 3600000 -v 3 -t 24 -u <YOUR_USERNAME>
```

#### Setting up a utility for participation in validator's election: 

For make pariticipation in validator's elections automatically we will use [ton-validation](https://github.com/everstake/ton-validation) utility.

Firstly: `git clone https://github.com/everstake/ton-validation.git && cd ton-validation`

```
# set env variable in .bashrc file using export, here user=ton
export FIFTPATH=/home/ton/ton-sources/ton/crypto/fift/lib:/home/ton/ton-sources/ton/crypto/smartcont
export BETTER_EXCEPTIONS=1
sudo yum -y install python3-pip
source env/bin/activate
#After that your promt will change
#Place requirements.txt and validator.py in directory with executables
#Before run next step you need to remove from 'requirements.txt' that line: 'pkg-resources==0.0.0'
pip install -r requirements.txt
#Run and check the output
python validator.py
#To exit run
deactivate
```

After that you need to copy into this directory (ton-validation) these files:

- build/lite-client/lite-client
- build/utils/generate-random-id
- build/validator-engine/validator-engine
- build/validator-engine-console/validator-engine-console
- build/crypto/fift
- client.pub (one of the keyrings, generated on the previous step)
- server.pub (one of the keyrings, generated on the previous step)
- liteserver.pub (one of the keyrings, generated on the previous step)

Then you need to create your validator wallet. You can do that with one of default .fift files, like: `new-wallet.fif`, `new-wallet-v2.fif`, etc. You can find them in that directory: `ton/crypto/smartcont`. 

Firstly create your new account's initialization (.boc) file:

`fift -s new-wallet.fif 0 `  -> result of this command: `new-wallet-query.boc`, `new-wallet.addr`, `new-wallet.pk`. You need to copy `.pk` and `.addr` files into `ton-validation` directory.

After someone send to your new wallet some GRMs, you need to open your `lite-client`  and execute:

`sendfile new-wallet-query.boc` -> if this query will be successful, your account will be initialized in that moment and you will be able to use it in any time in future, with your `new-wallet.pk`  (private key).

The last file, that you need to copy into `ton-validation` directory is `return-stake.boc`. You can get it, after running of this command: `fift -s recover-stake.fif`.

If you done all steps before rightly, you will get structure of your `ton-validation`  directory something like this:

![](https://habrastorage.org/webt/jc/he/lg/jchelg80zfvdi5sj6qiawbayuyo.png)

#### Configuration of your validator.py:

The last step is to configure rightly your `validator.py` script. For make it done you must open this script in any text editor, find the next lines and replace default values for your own:

1. `CONNECT_STR_LITE_CLIENT = "<YOUR_VALIDATOR_IP>:<YOUR_PORT_FOR_LITECLIENT_CONNECTION>"` (port for this line you can get from `/var/ton-work/db/config.json`);
2. `CONNECT_STR_ENGINE_CONSOLE = "<YOUR_VALIDATOR_IP>:<YOUR_PORT_FOR_VALIDATOR_CONSOLE_CONNECTION>"`;
3. `WALLET_ADDR = "<YOUR_WALLET_ADDR_NON_BOUNCABLE>"`;
4. `WALLET_ADDR_C = "d4eefbc9a5fbb4e98531f40c5c0b4e8de35fe86a494c3666520541a37fb7e7d3"`: here is your full wallet address, for example -1:d4eefbc9a5fbb4e98531f40c5c0b4e8de35fe86a494c3666520541a37fb7e7d3;
5. `WALLET_FILENAME = "new-wallet"`: the name of your wallet's private key file, without '.pk';
6. `MAX_FACTOR = "10"`: max_factor for your validator participation queries, for example 10 is default value, for our test network;
7. `STAKE = "<YOUR_VALIDATOR_STAKE>"`: for example, right now, in our testnet default working stake is 40001.

That's all! Other constants in file you shouldn't change.

If you done every step before, rightly, then you can run your validator.py like this:

```bash
source env/bin/activate
python validator.py
```

This script will do everything, related to your validator automatically: it will send query for participation in validators election (if them will be anounced), it will recover your stake with your bounties, etc.

#### Cron configuration:

If your `validator.py` script working without any errors, the last step that you need is setting up auto start of it. You can use for it your cron manager. For edit cron jobs on your server you can run: `crontab -e` and then add a new job:

```bash
*/3 * * * * cd /home/<YOUR_USERNAME>/ton-validation && env/bin/python validator.py > /dev/null 2>&1
```

Right now your full node is working fine and `validator.py` is starting every 3 minutes to make new queries for pariticipation in every new validation rounds.

**That's all!**