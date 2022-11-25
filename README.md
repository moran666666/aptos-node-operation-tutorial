<h1 align="center">Aptos Node Operation Tutorial of Source Code</h1>

# part I Setup

## 1. Hardware
* CPU: AMD EPYC 7302P
* MEM: 64G
* DISK: 1.96T NVMe SSD
* OS: ubuntu20.04
* USER: root

## 2. OS Environment
* Update system depends and install some package
```shell
sudo apt update
sudo apt upgrade -y
sudo apt install git unzip wget make gcc tmux jq lld -y
```

* Modify file handles number to 1048576
```shell
echo "ulimit -SHn 1048576" |  sudo tee -a /etc/profile
echo "* hard nofile 1048576" |  sudo tee -a /etc/security/limits.conf
echo "* soft nofile 1048576" |  sudo tee -a /etc/security/limits.conf
echo "DefaultLimitNOFILE=1048576" |  sudo tee -a /etc/systemd/user.conf
echo "DefaultLimitNOFILE=1048576" |  sudo tee -a /etc/systemd/system.conf
echo "session required pam_limits.so" |  sudo tee -a /etc/pam.d/common-session
source /etc/profile
```

## 3. Compile Source Binary Files
```shell
mkdir -p ~/aptos                                         # create work directory
cd ~/aptos                                               # Enter directory
git clone https://github.com/aptos-labs/aptos-core.git   # Down aptos project source code
cd aptos-core/                                           # Enter aptos project directory,ready to compile
./scripts/dev_setup.sh                                   # The first compilation needs to be executed, and can be omitted later
source ~/.cargo/env                                      # The first compilation needs to be executed, and can be omitted later
git checkout cf280b9a9013dddaeda359174b90236b60990533    # Switch to the official commit branch
cargo build -p aptos --release                           # Compile aptos binary files
cargo build -p aptos-node --release                      # Compile aptos-node binary files(cargo build --release compiled "apt os node" binary file may not be available)
sudo mv ./target/release/aptos* /usr/local/bin/          # Move the aptos and aptos-node binary files to the global bin directory
```

## 4. Set Global Environment Variables
* Through owner_address gets the operator's pool_address
```shell
aptos node get-stake-pool --owner-address <owner_address> --url https://fullnode.mainnet.aptoslabs.com/v1
```

* Set environment variables
```shell
echo "export USER_NAME=moran666666" |  sudo tee -a /etc/profile                                # Change to your own name
echo "export WORKSPACE=~/aptos" |  sudo tee -a /etc/profile                                    # Change to your own path (note that there should be no more "/" at the end)
echo "export DATA_DIR=/opt/aptos/data" |  sudo tee -a /etc/profile                             # Same with up
echo "export NODE_CFG_DIR=/opt/aptos/genesis" |  sudo tee -a /etc/profile                      # Same with up
echo "export VALIDATOR_IP=102.111.108.151" |  sudo tee -a /etc/profile                         # Change to your own validator ip
echo "export FULLNODE_IP=100.101.118.131" |  sudo tee -a /etc/profile                          # Change to your own vfn ip
echo "export POOL_ADDRESS=18a3ef48xxxxxxxxxxxxxxxxxxxxxxxxxxxxx" |  sudo tee -a /etc/profile   # Change to your own pool_address
echo "export REST_URL=https://fullnode.mainnet.aptoslabs.com/v1" |  sudo tee -a /etc/profile   # *Don't Change*
source /etc/profile
```

## 5. Down 2 Files genesis.blob、waypoint.txt
```shell
sudo wget https://raw.githubusercontent.com/aptos-labs/aptos-genesis-waypoint/main/mainnet/genesis.blob -O $NODE_CFG_DIR/genesis.blob
sudo wget https://raw.githubusercontent.com/aptos-labs/aptos-genesis-waypoint/main/mainnet/waypoint.txt -O $NODE_CFG_DIR/waypoint.txt
```
Remember to use sha256sum $NODE_CFG_DIR/genesis.blob compare hash value

## 6. Generate Some Key Pairs
* Generate key file
```shell
aptos genesis generate-keys --output-dir $WORKSPACE/keys
```
Will generate $WORKSPACE/keys directory，it includes: public-keys.yaml、private-keys.yaml、validator-identity.yaml、validator-full-node-identity.yaml, used for account,consensus communication...

* Copy the required files to the configuration directory $NODE_CFG_DIR
```shell
sudo mkdir -p $NODE_CFG_DIR
sudo cp $WORKSPACE/keys/validator-identity.yaml $NODE_CFG_DIR/
sudo cp $WORKSPACE/keys/validator-full-node-identity.yaml $NODE_CFG_DIR/
sudo cp $WORKSPACE/aptos-core/docker/compose/aptos-node/validator.yaml $NODE_CFG_DIR/
sudo cp $WORKSPACE/aptos-core/docker/compose/aptos-node/fullnode.yaml $NODE_CFG_DIR/
```

## 7. Modify Yaml Configuration Files
```shell
# validator
sudo sed -i "/.*account_address*/c\account_address: $POOL_ADDRESS" $NODE_CFG_DIR/validator-identity.yaml
sudo sed -i "s|\/opt\/aptos\/data|$DATA_DIR|g" $NODE_CFG_DIR/validator.yaml
sudo sed -i "s|\/opt\/aptos\/genesis|$NODE_CFG_DIR|g" $NODE_CFG_DIR/validator.yaml
sudo sed -i 's|"/ip4/0.0.0.0/tcp/6181"|"/ip4/127.0.0.1/tcp/16181"|g' $NODE_CFG_DIR/validator.yaml
sudo sed -i 's|"0.0.0.0:8080"|"127.0.0.1:18080"|g' $NODE_CFG_DIR/validator.yaml
sudo sed -i '/validator_network:/a\  listen_address: "/ip4/127.0.0.1/tcp/16180"' $NODE_CFG_DIR/validator.yaml

sudo sed -i '$s/$/\n/' $NODE_CFG_DIR/validator.yaml
sudo sed -i '$a\inspection_service:' $NODE_CFG_DIR/validator.yaml
sudo sed -i '$a\  address: "127.0.0.1"' $NODE_CFG_DIR/validator.yaml
sudo sed -i '$a\  port: 19101' $NODE_CFG_DIR/validator.yaml


# full node
sudo sed -i "/.*account_address*/c\account_address: $POOL_ADDRESS" $NODE_CFG_DIR/validator-full-node-identity.yaml
sudo sed -i "s|\/opt\/aptos\/data|$DATA_DIR|g" $NODE_CFG_DIR/fullnode.yaml
sudo sed -i "s|\/opt\/aptos\/genesis|$NODE_CFG_DIR|g" $NODE_CFG_DIR/fullnode.yaml
sudo sed -i "s|<Validator IP Address>|$VALIDATOR_IP|g" $NODE_CFG_DIR/fullnode.yaml
sudo sed -i 's|"/ip4/0.0.0.0/tcp/6182"|"/ip4/127.0.0.1/tcp/16182"|g' $NODE_CFG_DIR/fullnode.yaml
sudo sed -i 's|"0.0.0.0:8080"|"127.0.0.1:18080"|g' $NODE_CFG_DIR/fullnode.yaml

sudo sed -i '$s/$/\n/' $NODE_CFG_DIR/fullnode.yaml
sudo sed -i '$a\inspection_service:' $NODE_CFG_DIR/fullnode.yaml
sudo sed -i '$a\  address: "127.0.0.1"' $NODE_CFG_DIR/fullnode.yaml
sudo sed -i '$a\  port: 19101' $NODE_CFG_DIR/fullnode.yaml
```

## 8. Start Running Node
```shell
# validator
tmux new-window -t aptos -n node
tmux send-keys -t aptos:node "aptos-node -f $NODE_CFG_DIR/validator.yaml 2>&1 | tee ~/validator.log " C-m

# full node
tmux new-window -t aptos -n node
tmux send-keys -t aptos:node "aptos-node -f $NODE_CFG_DIR/fullnode.yaml 2>&1 | tee ~/fullnode.log " C-m
```
So far, the node can run, but the number of blocks cannot be synchronized because it is not in the validator node set (see step 11 for details). At the same time, for security, you need to run haproxy as a proxy

## 9. Setup And Start Haproxy
* setup haproxy
```shell
sudo apt install haproxy
haproxy -v
```

* Down and modify haproxy configuration file
```shell
# validator
sudo sed -i '$a\127.0.0.1 validator' /etc/hosts
sudo mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
sudo wget https://github.com/aptos-labs/aptos-core/raw/mainnet_haproxy/docker/compose/aptos-node/blocked.ips -O /etc/haproxy/blocked.ips
sudo wget https://github.com/aptos-labs/aptos-core/raw/mainnet_haproxy/docker/compose/aptos-node/haproxy.cfg -O /etc/haproxy/haproxy.cfg
sudo sed -i 's|bind :8180|bind :8080|g' /etc/haproxy/haproxy.cfg
sudo sed -i 's|server validator validator:6180|server validator validator:16180|g' /etc/haproxy/haproxy.cfg
sudo sed -i 's|server validator validator:6181|server validator validator:16181|g' /etc/haproxy/haproxy.cfg
sudo sed -i 's|server validator validator:8080|server validator validator:18080|g' /etc/haproxy/haproxy.cfg
sudo sed -i 's|server validator validator:9101|server validator validator:19101|g' /etc/haproxy/haproxy.cfg
sudo sed -i 's|/usr/local/etc/haproxy/blocked.ips|/etc/haproxy/blocked.ips|g' /etc/haproxy/haproxy.cfg
sudo sed -i '$a\\n' /etc/haproxy/haproxy.cfg
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.validator

# full node
sudo sed -i '$a\127.0.0.1 fullnode' /etc/hosts
sudo mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
sudo wget https://github.com/aptos-labs/aptos-core/raw/mainnet_haproxy/docker/compose/aptos-node/blocked.ips -O /etc/haproxy/blocked.ips
sudo wget https://github.com/aptos-labs/aptos-core/raw/mainnet_haproxy/docker/compose/aptos-node/haproxy-fullnode.cfg -O /etc/haproxy/haproxy.cfg
sudo sed -i 's|server fullnode fullnode:6182|server fullnode fullnode:16182|g' /etc/haproxy/haproxy.cfg
sudo sed -i 's|server fullnode fullnode:8080|server fullnode fullnode:18080|g' /etc/haproxy/haproxy.cfg
sudo sed -i 's|server fullnode fullnode:9101|server fullnode fullnode:19101|g' /etc/haproxy/haproxy.cfg
sudo sed -i 's|/usr/local/etc/haproxy/blocked.ips|/etc/haproxy/blocked.ips|g' /etc/haproxy/haproxy.cfg
sudo sed -i '$a\\n' /etc/haproxy/haproxy.cfg
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.fullnode
```

* Restart haproxy
```shell
sudo systemctl restart haproxy
```

* View the log of haproxy
```shell
sudo journalctl -u haproxy.service --since today --no-pager
```

## 10. Add To Validator Set
* Initialize Catalog File
```shell
cd $WORKSPACE
aptos init --profile mainnet-operator --private-key <account_private_key> --rest-url $REST_URL --skip-faucet
```
account_private_key: $WORKSPACE/keys/private-keys.yaml file "account_private_key" field

* Generate node information required on the chain<br>
Will generate $WORKSPACE/$USER_NAME directory，it includes: operator.yaml、owner.yaml, used for account,consensus communication...

```shell
aptos genesis set-validator-configuration \
    --local-repository-dir $WORKSPACE \
    --username $USER_NAME \
    --owner-public-identity-file $WORKSPACE/keys/public-keys.yaml \
    --validator-host validator.aptos.xyz:6180 \
    --full-node-host fullnode.aptos.xyz:6182 \
    --stake-amount 100000000000000
```
--validator-host: Change to own validator domain name + 6180<br>
--full-node-host: Change to own vfn domain name + 6182<br>
--stake-amount Change to own stake quantity(eight zeros are decimal precision)

* Join the validator set
```shell
# Update validator network address on chain
aptos node update-validator-network-addresses --pool-address $POOL_ADDRESS --operator-config-file $WORKSPACE/$USER_NAME/operator.yaml --profile mainnet-operator

# Update validator consensus key on chain
aptos node update-consensus-key --pool-address $POOL_ADDRESS --operator-config-file $WORKSPACE/$USER_NAME/operator.yaml --profile mainnet-operator

# Join to validator set
aptos node join-validator-set --pool-address $POOL_ADDRESS --profile mainnet-operator
```

* Check whether join the validator set is successful
```shell
aptos node show-validator-set --profile mainnet-operator | jq -r '.Result.pending_active' | grep -A10 $POOL_ADDRESS
aptos node show-validator-set --profile mainnet-operator | jq -r '.Result.active_validators' | grep -A10 $POOL_ADDRESS
```
After joined, your validator node information will first appear in pending_active, the next epoch will be appear to the active_validators

* Other useful commands for query
```shell
# Query number of proposals since run
curl 127.0.0.1:19101/metrics 2> /dev/null | grep "aptos_consensus_proposals_count"

# Query number of node connections
curl 127.0.0.1:19101/metrics 2> /dev/null | grep "aptos_connections{"

# Query block data sync info
curl 127.0.0.1:19101/metrics 2> /dev/null | grep "aptos_state_sync_version{type="

# Query the binary running version
curl 127.0.0.1:18080/v1 2> /dev/null | jq

# Query the current epoch proposal
aptos node analyze-validator-performance --analyze-mode=detailed-epoch-table "--url=http://127.0.0.1:18080" --start-epoch=-1 | grep $POOL_ADDRESS
```

## 11. Leave Validator Set
```shell
aptos node leave-validator-set --profile mainnet-operator --pool-address $POOL_ADDRESS
```

# part II Automatic Failover (unverified)

## 1. Logic Principle<br>
When the validator node fails, the vfn node is used for temporary emergency recovery:<br>
1). Change DNS from $VALIDATOR_IP to $FULLNODE_IP<br>
2). Stop the aptos-node process on the vfn node machine (aptos-node -f $NODE_CFG_DIR/fullnode.yaml)<br>
3). Switch the configuration file of haproxy to validator's file<br>
4). Restart haproxy: sudo systemctl restart haproxy<br>
5). Running validator on the vfn machine(aptos-node -f $NODE_CFG_DIR/validator.yaml); For the first time, when the vfn node changed to validator node and run, it may exit with an error. It can be solved by simply running again

## 2. Automation Script<br>
Save the following script to $WORKSPACE/auto_migrate.sh
```shell
#!/bin/bash

# 0. Modify the following variable contents to your own
dns_name="validator.aptos.xyz"
zone_id="09a3xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
token="PEzX4Txxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
update_ip=$FULLNODE_IP
validator_ip=$VALIDATOR_IP

# Judge validator node running information. If it's normal, the remaining scripts will not be executed and exit
chain_id=$(curl $validator_ip:8180/v1 2> /dev/null | jq -r ".chain_id")
if [[ $chain_id -eq 1 ]];then
    exit
fi

# Judge node machine is alive. If it's alive, the remaining scripts will not execute
while true
do
    active_count=$(ping validator_ip -c 1 | grep "ttl" | grep "time" | wc -l)
    if [[ $active_count -ge 1 ]];then
        exit
    fi
    sleep 3
done

# 1. update domain name
dns_id=$(curl -X GET "https://api.cloudflare.com/client/v4/zones/$zone_id/dns_records?name=$dns_name&type=A" \
          -H "Authorization: Bearer $token" \
          -H "Content-Type:application/json" \
          2> /dev/null | jq -r ".result[0].id")
echo "dns_id: $dns_id"

update_result=$(curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/$zone_id/dns_records/$dns_id" \
     -H "Authorization: Bearer $token" \
     -H "Content-Type: application/json" \
     --data "{\"type\":\"A\",\"name\":\"$dns_name\",\"content\": \"$update_ip\", \"ttl\":1,\"proxied\":false}")
echo $update_result | jq -r ".success"

# 2. Send the abort program signal to aptos:node pane of tmux
while true
do
    tmux send-keys -t aptos:node 'C-c' C-m
    sudo pkill -9 aptos-node
    sleep 3
    process_info=$(ps -aux | grep -v color=auto | grep aptos-node)
    if [[ -z "$process_info" ]];then
        break
    fi
done

# 3. switch haproxy.cfg file
sudo rm -rf /etc/haproxy/haproxy.cfg
sudo cp /etc/haproxy/haproxy.cfg.validator /etc/haproxy/haproxy.cfg

# 4. restart haproxy
sudo systemctl restart haproxy

# 5. Send the run validator node command to aptos:node pane of tmux
while true
do
    tmux send-keys -t aptos:node "aptos-node -f $NODE_CFG_DIR/validator.yaml 2>&1 | tee ~/validator.log " C-m
    sleep 30
    process_info=$(ps -aux | grep -v color=auto | grep aptos-node)
    if [[ $process_info ]];then
        break
    fi
done
```

Run the above script file as a task every 3 minutes:
```shell
chmod +x $WORKSPACE/auto_migrate.sh

crontab -e
*/3 * * * * cd $WORKSPACE && bash auto_migrate.sh

```

## 3. cloudflare Domain Name API Settings<br>
First, you need to have a cloudfare account and a domain name, then log in and jump to https://dash.cloudflare.com/profile/api-tokens set the API. See the following figure for specific settings. We only need the Token and Zone_ID.

![1](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/creat_token_1.jpg)
![2](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/creat_token_2.jpg)
![3](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/creat_token_3.jpg)
![4](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/creat_token_4.jpg)
![5](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/creat_token_5.jpg)
![6](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/creat_token_6.jpg)
![7](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/api_zoneid.jpg)
