# Run a Node

The below are instructions for running on the public Lilypad testnet.

## Adding a node

The testnet has a base currency of ETH, as well as a utility token called LP. You will receive LP to pay for jobs (and nodes to stake).

### Metamask
We suggest using Metamask with custom settings to make things easier. Once you have it installed and setup, here are the settings you should use:
- Network name: Lilypad v2 Aurora testnet
- New RPC URL: [http://testnet.lilypad.tech:8545](http://testnet.lilypad.tech:8545)
- Chain ID: 1337
- Currency symbol: ETH
- Block explorer URL: (leave blank)

### Fund your wallet with ETH and LP
To obtain funds, go to [http://faucet.lilypad.tech:8080](http://faucet.lilypad.tech:8080)

The faucet will give you both ETH (to pay for gas) and LP (to stake and pay for jobs).

## Prerequisites
- Linux (latest Ubuntu LTS recommended)
- Nvidia GPU
- Nvidia drivers
- [Docker using this guide](https://docs.docker.com/engine/install/ubuntu/). Note: do not run `apt-get install` as it will install the GUI version. 
- Nvidia docker drivers

### Install Bacalhau
```
cd /tmp
wget https://github.com/bacalhau-project/bacalhau/releases/download/v1.0.3/bacalhau_v1.0.3_linux_amd64.tar.gz
tar xfv bacalhau_v1.0.3_linux_amd64.tar.gz
sudo mv bacalhau /usr/bin/bacalhau
sudo mkdir -p /app/data/ipfs
sudo chown -R $USER /app/data
```

### Install Lilypad
```
curl -sSL -o lilypad https://github.com/bacalhau-project/lilypad/releases/download/v2.0.0-d63a7ff/lilypad-linux-amd64
chmod +x lilypad
sudo mv lilypad /usr/bin/lilypad
```

### Write env file
You will need to create an environment file for your node.
`/app/lilypad/resource-provider-gpu.env` should contain:
```
WEB3_PRIVATE_KEY=YOUR_PRIVATE_KEY (the private key from a NEW metamask wallet FOR THE COMPUTE NODE)
```

This is the key where you will get paid in LP tokens for jobs run on the network.

**NOTE: YOU MUST NOT REUSE YOUR COMPUTE NODE KEY AS A CLIENT, EVEN FOR TESTING: THIS WILL RESULT IN FAILED JOBS AND WILL NEGATIVELY IMPACT YOUR COMPUTE NODE SINCE THE WALLET ADDRESS IS HOW NODES ARE IDENTIFIED ON THE NETWORK**

### Install systemd unit for Bacalhau:
Open `/etc/systemd/system/bacalhau.service` in your favourite editor (hint: `sudo editor /etc/systemd/system/bacalhau.service`)
```
[Unit]
Description=Lilypad V2 Bacalhau
After=network-online.target
Wants=network-online.target systemd-networkd-wait-online.service

[Service]
Environment="LOG_TYPE=json"
Environment="LOG_LEVEL=debug"
Environment="HOME=/app/lilypad"
Environment="BACALHAU_SERVE_IPFS_PATH=/app/data/ipfs"
Restart=always
RestartSec=5s
ExecStart=/usr/bin/bacalhau serve --node-type compute,requester --peer none --private-internal-ipfs=false

[Install]
WantedBy=multi-user.target
```

### Install systemd unit for GPU provider
Open `/etc/systemd/system/lilypad-resource-provider.service` in your favourite editor (hint: `sudo editor /etc/systemd/system/lilypad-resource-provider.service`)
```
[Unit]
Description=Lilypad V2 Resource Provider GPU
After=network-online.target
Wants=network-online.target systemd-networkd-wait-online.service

[Service]
Environment="LOG_TYPE=json"
Environment="LOG_LEVEL=debug"
Environment="HOME=/app/lilypad"
Environment="OFFER_GPU=1"
EnvironmentFile=/app/lilypad/resource-provider-gpu.env
Restart=always
RestartSec=5s
ExecStart=/usr/bin/lilypad resource-provider 

[Install]
WantedBy=multi-user.target
```

Reload systemd's units/daemons (you will need to do this again if you ever change the systemd unit files that we wrote, above)
```
sudo systemctl daemon-reload
```

Start systemd units:
```
sudo systemctl start bacalhau
sudo systemctl start lilypad-resource-provider
```

Check they are running with `systemctl` status as usual, and debug with `journalctl` if needed - eg, `sudo journalctl -uf lilypad-resource-provider` will give you the live output from your Lilypad node. Please report issues on Bacalhau `#lilypad-general` Slack. You should see your resource provider start accepting jobs on the network in the logs.

### Security
If you want to allowlist only certain modules (e.g. Stable Diffusion modules), so that you can control exactly what code runs on your nodes (which you can audit to ensure that they are secure and will have no negative impact on your nodes), you can do that by setting an environment variable `OFFER_MODULES` in the GPU provider to a comma separated list of module names, e.g. “sdxl:v0.9-lilypad1,stable-diffusion:v0.0.1”

See https://github.com/bacalhau-project/lilypad/#available-modules for a list of modules.
