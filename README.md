Dependencies Installation

**Install dependencies for building from source**
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

**Install Go**
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
Node Installation
```


**Clone project repository**
```
cd && rm -rf elys
git clone https://github.com/elys-network/elys
cd elys
git checkout v0.51.0
```

**Build binary**
```
make install
```

**Prepare cosmovisor directories**
```
mkdir -p $HOME/.elys/cosmovisor/genesis/bin
ln -s $HOME/.elys/cosmovisor/genesis $HOME/.elys/cosmovisor/current -f
```

**Copy binary to cosmovisor directory**
```
cp $(which elysd) $HOME/.elys/cosmovisor/genesis/bin
```

**Set node CLI configuration**
```
elysd config chain-id elystestnet-1
elysd config keyring-backend test
elysd config node tcp://localhost:22057
```

**Initialize the node**
```
elysd init "Your Node Name" --chain-id elystestnet-1
```

**Download genesis and addrbook files**
```
curl -L https://snapshots-testnet.nodejumper.io/elys/genesis.json > $HOME/.elys/config/genesis.json
curl -L https://snapshots-testnet.nodejumper.io/elys/addrbook.json > $HOME/.elys/config/addrbook.json
```

**Set seeds**
```
sed -i -e 's|^seeds *=.*|seeds = "cdf9ae8529aa00e6e6703b28f3dcfdd37e07b27c@37.187.154.66:26656,86987eeff225699e67a6543de3622b8a986cce28@91.183.62.162:26656,ae22b82b1dc34fa0b1a64854168692310f562136@198.27.74.140:26656,61284a4d71cd3a33771640b42f40b2afda389a1e@5.101.138.254:26656,ae7191b2b922c6a59456588c3a262df518b0d130@elys-testnet-seed.itrocket.net:54656,8542cd7e6bf9d260fef543bc49e59be5a3fa9074@seed.publicnode.com:26656,609c64cc50fb4ebbe7cae3347545d3950ea2c018@65.108.195.29:23656,0977dd5475e303c99b66eaacab53c8cc28e49b05@elys-testnet-peer.itrocket.net:38656"|' $HOME/.elys/config/config.toml
```

**Set minimum gas price**
```
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.01uelys,0.01ibc/2180E84E20F5679FCC760D8C165B60F42065DEF7F46A72B447CFF1B7DC6C0A65,0.01ibc/E2D2F6ADCC68AA3384B2F5DFACCA437923D137C14E86FB8A10207CF3BED0C8D4"|' $HOME/.elys/config/app.toml
```

**Set pruning**
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.elys/config/app.toml
```

**Enable prometheus**
```
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.elys/config/config.toml
```

**Change ports**
```
sed -i -e "s%:1317%:22017%; s%:8080%:22080%; s%:9090%:22090%; s%:9091%:22091%; s%:8545%:22045%; s%:8546%:22046%; s%:6065%:22065%" $HOME/.elys/config/app.toml
sed -i -e "s%:26658%:22058%; s%:26657%:22057%; s%:6060%:22060%; s%:26656%:22056%; s%:26660%:22061%" $HOME/.elys/config/config.toml
```

**Download latest chain data snapshot**
```
curl "https://snapshots-testnet.nodejumper.io/elys/elys_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.elys"
```

**Install Cosmovisor**
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0
```

**Create a service**
```
sudo tee /etc/systemd/system/elys.service > /dev/null << EOF
[Unit]
Description=Elys Network node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.elys
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.elys"
Environment="DAEMON_NAME=elysd"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable elys.service
```

**Start the service and check the logs**
```
sudo systemctl start elys.service
sudo journalctl -u elys.service -f --no-hostname -o cat
Secure Server Setup (Optional)
```

**generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE**
```
ssh-keygen -t rsa
```

**save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY**
```
cat ~/.ssh/id_rsa.pub
```

**upgrade system packages**
```
sudo apt update
sudo apt upgrade -y
```

**add new admin user**
```
sudo adduser admin --disabled-password -q
```

**upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above**
```
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
```

**disable root login, disable password authentication, use ssh keys only**
```
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd
```

**install fail2ban**
```
sudo apt install -y fail2ban
```

**install and configure firewall**
```
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656
```

**make sure you expose ALL necessary ports, only after that enable firewall**
```
sudo ufw enable
```

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
