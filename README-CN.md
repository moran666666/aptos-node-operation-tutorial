<h1 align="center">aptos节点源码运维教程</h1>

# part I 安装运行

## 1. 硬件情况
* CPU: AMD EPYC 7302P
* 内存: 64G
* 硬盘: 1.96T NVMe SSD
* 操作系统: ubuntu20.04
* 操作用户: root

## 2. 系统环境
* 更新系统依赖及安装用到的软件
```shell
sudo apt update
sudo apt upgrade -y
sudo apt install git unzip wget make gcc tmux jq lld -y
```

* 更新系统的文件句柄数量为1048576
```shell
echo "ulimit -SHn 1048576" |  sudo tee -a /etc/profile
echo "* hard nofile 1048576" |  sudo tee -a /etc/security/limits.conf
echo "* soft nofile 1048576" |  sudo tee -a /etc/security/limits.conf
echo "DefaultLimitNOFILE=1048576" |  sudo tee -a /etc/systemd/user.conf
echo "DefaultLimitNOFILE=1048576" |  sudo tee -a /etc/systemd/system.conf
echo "session required pam_limits.so" |  sudo tee -a /etc/pam.d/common-session
source /etc/profile
```

## 3. 编译源码二进制文件
```shell
mkdir -p ~/aptos                                         # 创建工作目录
cd ~/aptos                                               # 进入工作目录
git clone https://github.com/aptos-labs/aptos-core.git   # 下载aptos项目源码
cd aptos-core/                                           # 进入项目源码目录,准备编译
./scripts/dev_setup.sh                                   # 第二次之后的编译可省略执行此命令
source ~/.cargo/env                                      # 第二次之后的编译可省略执行此命令
git checkout cf280b9a9013dddaeda359174b90236b60990533    # 切换到官方要求的commit分支
cargo build -p aptos --release                           # 编译 aptos 二进制
cargo build -p aptos-node --release                      # 编译 aptos-node 二进制(cargo build --release 编译的“aptos-node”二进制文件可能会无法使用)
sudo mv ./target/release/aptos* /usr/local/bin/          # 将编译出来的aptos, aptos-node移动到全局的bin目录下，方便使用
```

## 4. 设置常用的全局环境变量
* 通过 owner_address 地址获取操作员的 pool_address
```shell
aptos node get-stake-pool --owner-address <owner_address> --url https://fullnode.mainnet.aptoslabs.com/v1
```

* 设置环境变量
```shell
echo "export USER_NAME=moran666666" |  sudo tee -a /etc/profile                                # 更改为自己的名称
echo "export WORKSPACE=~/aptos" |  sudo tee -a /etc/profile                                    # 更改为自己的路径(注意最后面不要多了"/")
echo "export DATA_DIR=/opt/aptos/data" |  sudo tee -a /etc/profile                             # 更改为自己的路径(注意最后面不要多了"/")
echo "export NODE_CFG_DIR=/opt/aptos/genesis" |  sudo tee -a /etc/profile                      # 更改为自己的路径(注意最后面不要多了"/")
echo "export VALIDATOR_IP=102.111.108.151" |  sudo tee -a /etc/profile                         # 更改为自己的验证节点ip
echo "export FULLNODE_IP=100.101.118.131" |  sudo tee -a /etc/profile                          # 更改为自己的全节点ip
echo "export POOL_ADDRESS=18a3ef48xxxxxxxxxxxxxxxxxxxxxxxxxxxxx" |  sudo tee -a /etc/profile   # 更改为自己的pool_address
echo "export REST_URL=https://fullnode.mainnet.aptoslabs.com/v1" |  sudo tee -a /etc/profile   # *不要更改*
source /etc/profile
```

## 5. 下载2个文件 genesis.blob、waypoint.txt
```shell
sudo wget https://raw.githubusercontent.com/aptos-labs/aptos-genesis-waypoint/main/mainnet/genesis.blob -O $NODE_CFG_DIR/genesis.blob
sudo wget https://raw.githubusercontent.com/aptos-labs/aptos-genesis-waypoint/main/mainnet/waypoint.txt -O $NODE_CFG_DIR/waypoint.txt
```
记得使用 sha256sum $NODE_CFG_DIR/genesis.blob 命令对比hash值是否一致，以防下载的文件不对

## 6. 生成各种密钥对
* 生成密钥文件
```shell
aptos genesis generate-keys --output-dir $WORKSPACE/keys
```
会生成$WORKSPACE/keys目录，目录下包括文件：public-keys.yaml、private-keys.yaml、validator-identity.yaml、validator-full-node-identity.yaml，用于帐户、共识通信用...

* 拷贝所需要的配置文件到配置目录$NODE_CFG_DIR
```shell
sudo mkdir -p $NODE_CFG_DIR
sudo cp $WORKSPACE/keys/validator-identity.yaml $NODE_CFG_DIR/
sudo cp $WORKSPACE/keys/validator-full-node-identity.yaml $NODE_CFG_DIR/
sudo cp $WORKSPACE/aptos-core/docker/compose/aptos-node/validator.yaml $NODE_CFG_DIR/
sudo cp $WORKSPACE/aptos-core/docker/compose/aptos-node/fullnode.yaml $NODE_CFG_DIR/
```

## 7. 修改配置文件
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

## 8. 运行节点
```shell
# validator
tmux new-window -t aptos -n node
tmux send-keys -t aptos:node "aptos-node -f $NODE_CFG_DIR/validator.yaml 2>&1 | tee ~/validator.log " C-m

# full node
tmux new-window -t aptos -n node
tmux send-keys -t aptos:node "aptos-node -f $NODE_CFG_DIR/fullnode.yaml 2>&1 | tee ~/fullnode.log " C-m
```
到目前为止,节点可以运行了,但还无法同步区块数，因为还没在验证者节点集中(详情看步骤11). 同时为了安全，还需要运行haproxy作为代理

## 9. 安装运行haproxy代理
* 安装haproxy
```shell
sudo apt install haproxy
haproxy -v
```

* 下载修改haproxy的配置文件
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

* 重启haproxy
```shell
sudo systemctl restart haproxy
```

* 查看haproxy的日志
```shell
sudo journalctl -u haproxy.service --since today --no-pager
```

## 10. 将节点加入验证器集
* 初始化目录文件
```shell
cd $WORKSPACE
aptos init --profile mainnet-operator --private-key <account_private_key> --rest-url $REST_URL --skip-faucet
```
account_private_key 为$WORKSPACE/keys/private-keys.yaml文件的account_private_key字段  

* 生成链上需要的节点信息<br>
会生成$WORKSPACE/$USER_NAME目录，目录下包括文件：operator.yaml、owner.yaml，用于帐户、共识通信...
```shell
aptos genesis set-validator-configuration \
    --local-repository-dir $WORKSPACE \
    --username $USER_NAME \
    --owner-public-identity-file $WORKSPACE/keys/public-keys.yaml \
    --validator-host validator.aptos.xyz:6180 \
    --full-node-host fullnode.aptos.xyz:6182 \
    --stake-amount 100000000000000
```
--validator-host 可以改为自己验证节点的域名+端口的方式<br>
--full-node-host 可以改为自己全节点的域名+端口的方式<br>
--stake-amount 可以改为自己有多少质押币数量，社区一般是官方给的100万(后面8个0是小数精度)

* 加入链上的验证器集
```shell
# 更新链上验证器网络地址
aptos node update-validator-network-addresses --pool-address $POOL_ADDRESS --operator-config-file $WORKSPACE/$USER_NAME/operator.yaml --profile mainnet-operator

# 更新链上验证器共识密钥
aptos node update-consensus-key --pool-address $POOL_ADDRESS --operator-config-file $WORKSPACE/$USER_NAME/operator.yaml --profile mainnet-operator

# 加入链上的验证器集
aptos node join-validator-set --pool-address $POOL_ADDRESS --profile mainnet-operator
```

* 检查加入验证器集是否成功
```shell
aptos node show-validator-set --profile mainnet-operator | jq -r '.Result.pending_active' | grep -A10 $POOL_ADDRESS
aptos node show-validator-set --profile mainnet-operator | jq -r '.Result.active_validators' | grep -A10 $POOL_ADDRESS
```
加入后,自己的验证节点信息先会出现在待激活集中pending_active,当下一个epoch到来时才会出现在激活集中active_validators.

* 其他有用的查询关键指标命令
```shell
# 查询运行以来自己的提案数量
curl 127.0.0.1:19101/metrics 2> /dev/null | grep "aptos_consensus_proposals_count"

# 查看节点连接数量
curl 127.0.0.1:19101/metrics 2> /dev/null | grep "aptos_connections{"

# 查看区块同步信息
curl 127.0.0.1:19101/metrics 2> /dev/null | grep "aptos_state_sync_version{type="

# 查看运行的二进制版本
curl 127.0.0.1:18080/v1 2> /dev/null | jq

# 查询当前epoch的提案情况
aptos node analyze-validator-performance --analyze-mode=detailed-epoch-table "--url=http://127.0.0.1:18080" --start-epoch=-1 | grep $POOL_ADDRESS
```

## 11. 离开验证器集
```shell
aptos node leave-validator-set --profile mainnet-operator --pool-address $POOL_ADDRESS
```

# part II 故障自动迁移(未验证)

## 1. 原理<br>
当验证节点validator发生故障时,利用全节点进行临时紧急恢复步骤:<br>
1). 将DNS由原来的 $VALIDATOR_IP 修改为 $FULLNODE_IP<br>
2). 停止全节点机器上的全节点aptos-node进程 (aptos-node -f $NODE_CFG_DIR/fullnode.yaml)<br>
3). 将haproxy的配置文件切换为验证节点的<br>
4). 重启haproxy: sudo systemctl restart haproxy<br>
5). 启动全节点机器上启动验证节点aptos-node (aptos-node -f $NODE_CFG_DIR/validator.yaml); 第一次由全节点书改为验证节点运行时可能会出现运行错误退出的情况，只需再次启动就能解决

## 2. 自动动化脚本<br>
将下面脚本保存至全节点机器的 $WORKSPACE/auto_migrate.sh
```shell
#!/bin/bash

# 0. 修改下面的变量内容，变更为自己的
dns_name="validator.aptos.xyz"
zone_id="09a3xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
token="PEzX4Txxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
update_ip=$FULLNODE_IP
validator_ip=$VALIDATOR_IP

# 判断验证节点运行情况，正常则余下的脚本不再执行直接退出脚本
chain_id=$(curl $validator_ip:8180/v1 2> /dev/null | jq -r ".chain_id")
if [[ $chain_id -eq 1 ]];then
    exit
fi

# 判断验证节点机器是否存活，存活则余下的脚本不再执行直接退出脚本
while true
do
    active_count=$(ping validator_ip -c 1 | grep "ttl" | grep "time" | wc -l)
    if [[ $active_count -ge 1 ]];then
        exit
    fi
    sleep 3
done

# 1. 更新域名
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

# 2. 向tmux的aptos:node窗格发送中止程序信号
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

# 3. 切换haproxy配置文件
sudo rm -rf /etc/haproxy/haproxy.cfg
sudo cp /etc/haproxy/haproxy.cfg.validator /etc/haproxy/haproxy.cfg

# 4. 重启haproxy
sudo systemctl restart haproxy

# 5. 向tmux的aptos:node窗格发送运行验证节点命令
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

将上面脚本文件，作为任务每隔3分钟运行一次:
```shell
chmod +x $WORKSPACE/auto_migrate.sh

crontab -e
*/3 * * * * cd $WORKSPACE && bash auto_migrate.sh

```

## 3. cloudflare域名API设置<br>
首先，自己要有cloudfare账号且存在域名,然后登陆跳转到 https://dash.cloudflare.com/profile/api-tokens 设置API,具体设置见下面图示,我们仅需要API的Token,Zone_ID这2个信息.

![1](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/creat_token_1.jpg)
![2](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/creat_token_2.jpg)
![3](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/creat_token_3.jpg)
![4](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/creat_token_4.jpg)
![5](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/creat_token_5.jpg)
![6](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/creat_token_6.jpg)
![7](https://github.com/moran666666/aptos-node-operation-tutorial/raw/main/api_zoneid.jpg)
