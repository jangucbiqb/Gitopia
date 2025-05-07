Hardware Requirements

Minimum

4CPU 8RAM 100GB
Recommended

8CPU 32RAM 200GB
Rent On Hetzner | Rent On OVH
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
cd && rm -rf gitopia
git clone https://github.com/gitopia/gitopia
cd gitopia
git checkout v5.1.0
```

# Build binary
make install

# Prepare cosmovisor directories
mkdir -p $HOME/.gitopia/cosmovisor/genesis/bin
ln -s $HOME/.gitopia/cosmovisor/genesis $HOME/.gitopia/cosmovisor/current -f

# Copy binary to cosmovisor directory
cp $(which gitopiad) $HOME/.gitopia/cosmovisor/genesis/bin

# Set node CLI configuration
gitopiad config chain-id gitopia
gitopiad config keyring-backend file
gitopiad config node tcp://localhost:11357

# Initialize the node
gitopiad init "Your Node Name" --chain-id gitopia

# Download genesis and addrbook files
curl -L https://snapshots.nodejumper.io/gitopia/genesis.json > $HOME/.gitopia/config/genesis.json
curl -L https://snapshots.nodejumper.io/gitopia/addrbook.json > $HOME/.gitopia/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "ade4d8bc8cbe014af6ebdf3cb7b1e9ad36f412c0@seeds.polkachu.com:11356,20e1000e88125698264454a884812746c2eb4807@seeds.lavenderfive.com:11356,ebc272824924ea1a27ea3183dd0b9ba713494f83@gitopia-mainnet-seed.autostake.com,187425bc3739daaef8cb1d7cf47d655117396dbe@seed-gitopia.ibs.team:16660,9aa8a73ea9364aa3cf7806d4dd25b6aed88d8152@gitopia-seed.mzonder.com:11056,400f3d9e30b69e78a7fb891f60d76fa3c73f0ecc@gitopia.rpc.kjnodes.com:14159,f280239045928af4e1b289d9df4059b7f941777b@seed-node.mms.team:35656,6d41d36d54abd868c4cdaf5b956ac047327bff67@seeds-3.anode.team:10260,8542cd7e6bf9d260fef543bc49e59be5a3fa9074@seed.publicnode.com:26656,08bc9afd0cac4ae6cf8f1877920b0cc7e58a6f42@seeds.tendermint.roomit.xyz:40001"|' $HOME/.gitopia/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.0012ulore"|' $HOME/.gitopia/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.gitopia/config/app.toml

# Enable prometheus
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.gitopia/config/config.toml

# Change ports
sed -i -e "s%:1317%:11317%; s%:8080%:11380%; s%:9090%:11390%; s%:9091%:11391%; s%:8545%:11345%; s%:8546%:11346%; s%:6065%:11365%" $HOME/.gitopia/config/app.toml
sed -i -e "s%:26658%:11358%; s%:26657%:11357%; s%:6060%:11360%; s%:26656%:11356%; s%:26660%:11361%" $HOME/.gitopia/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots.nodejumper.io/gitopia/gitopia_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.gitopia"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.0

# Create a service
sudo tee /etc/systemd/system/gitopia.service > /dev/null << EOF
[Unit]
Description=Gitopia node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.gitopia
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.gitopia"
Environment="DAEMON_NAME=gitopiad"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable gitopia.service

# Start the service and check the logs
sudo systemctl start gitopia.service
sudo journalctl -u gitopia.service -f --no-hostname -o cat
Secure Server Setup (Optional)

# generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE
ssh-keygen -t rsa

# save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY
cat ~/.ssh/id_rsa.pub
# upgrade system packages
sudo apt update
sudo apt upgrade -y

# add new admin user
sudo adduser admin --disabled-password -q

# upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# disable root login, disable password authentication, use ssh keys only
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd

# install fail2ban
sudo apt install -y fail2ban

# install and configure firewall
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
