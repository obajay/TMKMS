![securiy](https://github.com/obajay/TMKMS/assets/44331529/adeec44c-45e0-4072-8ce4-e74d2aa183a8)

<h1 align="center"> TMKMS</h1>

- The Tendermint Key Management System (or TMKMS) should be used by any validator currently or intending to be in the active validator set. This application mitigates the risk of double-signing and provides high-availability to validator keys while keeping these keys on a separate physical host. While TMKMS can be used on the same machine as the validator, it is recommended to be on a separate host.

# Let's look at the example of the SGE blockchain
[DOCS](https://docs.osmosis.zone/osmosis-core/keys/tmkms/)
=

## Install GCC and RUST
```python
sudo apt update
sudo apt install git build-essential ufw curl jq snapd --yes
curl --proto '=https' --tlsv1.3 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

## Create new user
```python
adduser tmkms
usermod -aG sudo tmkms
su tmkms && cd $HOME
```

## Compile TMKMS binaries
```python
cd $HOME
cargo install tmkms --features=softsign
sudo mv $HOME/.cargo/bin/tmkms /usr/local/bin/
```

## Create and Init TKMS working directory
```python
mkdir -p $HOME/tmkms/sge
tmkms init $HOME/tmkms/sge
```

## Import Private key
- upload your active priv_validator_key.json from your validator to directory `/home/tmkms/priv_validator_key.json`
```python
cat $HOME/priv_validator_key.json
tmkms softsign import $HOME/priv_validator_key.json $HOME/tmkms/sge/secrets/sge-consensus.key
sudo shred -uvz $HOME/priv_validator_key.json
```
## Swap tmkms.toml to the one below. The only "addr =" field edit need to be done, replace it with your validator node IP + port(26658 default)

```python
rm -rf ~/tmkms/sge/tmkms.toml

tee ~/tmkms/sge/tmkms.toml << EOF
#Tendermint KMS configuration file
[[chain]]
id = "sgenet-1"
key_format = { type = "bech32", account_key_prefix = "sgepub", consensus_key_prefix = "sgevalcons" }
state_file = "$HOME/tmkms/sge/state/sgenet-1_priv_validator_state.json"
#Software-based Signer Configuration
[[providers.softsign]]
chain_ids = ["sgenet-1"]
key_type = "consensus"
path = "$HOME/tmkms/sge/secrets/sge-consensus.key"
#Validator Configuration
[[validator]]
chain_id = "sgenet-1"
addr = "tcp://10.12.24.217:26658" #Set here IP and port of the SGE node you will be using for signing blocks (port can be custom)   
secret_key = "$HOME/tmkms/sge/secrets/kms-identity.key"
protocol_version = "v0.34"
reconnect = true
EOF
```

## Create service file and run TMKMS
```python
sudo tee /etc/systemd/system/tmkmsd-sge.service << EOF
[Unit]
Description=TMKMS-sge
After=network.target
StartLimitIntervalSec=0
[Service]
Type=simple
Restart=always
RestartSec=10
User=$USER
ExecStart=$(which tmkms) start -c $HOME/tmkms/sge/tmkms.toml
LimitNOFILE=1024
[Install]
WantedBy=multi-user.target
EOF
```
```pytohn
sudo systemctl daemon-reload
sudo systemctl enable tmkmsd-sge.service
sudo systemctl restart tmkmsd-sge.service
sudo systemctl status tmkmsd-sge.service
sudo journalctl -u tmkmsd-sge.service -f -o cat
```

`Its Normal`
![error](https://github.com/obajay/TMKMS/assets/44331529/03292c8c-5929-41dd-949b-c32a30cfe122)


- Now we need  go to our server (where is your validator located) and find field `priv_validator_laddr = ""` at dir `$HOME/.sge/config/config.toml` and edit to your Validator `IP + port`
- Example : `priv_validator_laddr = "tcp://0.0.0.0:26658"`

<details>
<summary>UFW</summary>
  
- Make sure your firewall open only for KMS server IP to allow connect to port 26658 (or any custom port u set)
```python
apt install ufw
ufw allow 22
ufw allow 80
ufw allow 443
ufw deny 26658 #deny access to this port
ufw allow from 55.19.14.470 #We allow access to our TMKMS server (specify your IP)
ufw enable
ufw status
```
</details>


### Restart your node
And make sure on the `tmkms` server that the logs are now correct
![logs](https://github.com/obajay/TMKMS/assets/44331529/7ff16c90-5d01-4c4a-a2ef-4e558c939042)

- Let's delete (after saving the file in a safe place) `priv_validator_key.json` from the validator node and restart again. **Everything should work**

<h1 align="center"> 📚Useful commands📚 </h1>

`logs`
```python
sudo journalctl -u tmkmsd-sge -f -o cat
```
`restart`
```python
sudo systemctl restart tmkmsd-sge && sudo journalctl -u tmkmsd-sge -f -o cat
```
