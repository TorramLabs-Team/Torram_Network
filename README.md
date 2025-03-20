# The Torram Network

TORRAM is building the rails for the future of Institutional DeFi and RWAs using Bitcoin as a natively secured settlement layer. 

By participating as a node validator, you help secure real-time financial infrastructure on Bitcoin — without the need for bridges, wrapped assets, or Layer 2s.

## Why run a Torram Testnet node?

- Exclusive early adopter benefits – only 30 testnet slots available
- Secure critical financial infrastructure directly on Bitcoin natively 
- Earn token rewards through blockchain production at mainnet 
- First node operators for a Bitcoin native oracle network enabling institutional use cases 
- No bridges, no wrapped BTC, no custodial risk

## Oracle Price Feeds

The initial set of testnet validators are expected to submit the following price feeds:

| Crypto | Full Name                          |
|--------|------------------------------------|
| BTC    | Bitcoin                            |
| ETH    | Ethereum                           |
| USDT   | Tether                             |
| USDC   | USD Coin                           |
| SOL    | Solana                             |
| DOGE   | Dogecoin                           |
| BNB    | Binance Chain                      |
| NEAR   | Near Protocol                      |
| DAI    | Dai                                |
| UNI    | Uniswap                            |
| TON    | Toncoin                            |
| ADA    | Cardano                            |
| OM     | Mantra                             |
| ONDO   | Ondo Finance                       |
| TRX    | Tron                               |
| AVAX   | Avalanche                          |
| AAVE   | Aave                               |
| STX    | Stacks                             |
| XLM    | Stellar                            |

| BRC20  | Full Name                          |
|--------|------------------------------------|
| ORDI   | Ordinals                           |
| MERL   | Merlin Chain                       |
| SATS   | Sats Coin                          |

| Runes     | Full Name                       |
|-----------|---------------------------------|
| LIQUIDIUM | Liqiudium Token                 |
| PUPS      | Pups World Peace                |
| DOG       | Dog Go To The Moon              |

| Index  | Full Name                          |
|--------|------------------------------------|
| DJI    | Dow Jones Industrial Average       |
| BAI    | Blackrock AI Innovation and Tech   |
| FTBFX  | Fidelity Total Bond Fund           |
| FVX    | 5-Year Treasury Yield              |
| NDAQ   | Nasdaq Composite                   |
| QQQ    | Invesco QQQ                        |
| RUT    | Russell 2000 Index                 |
| SPX    | S&P 500 Index                      |
| TNX    | 10-Year Treasury Yield             |
| TYX    | 30-Year Treasury Yield             |
| VGT    | Vanguard Information Technology    |
| VIX    | Volatility Index                   |


VWAP prices are calculated and stored in a Torram block every ~1 minute from these data sources. This is a default configuration currently, and there is no action required from Torram validators.

Prices are sourced from the following: 

**Crypto**
- Binance
- Coinbase
- Kraken
- KuCoin
- OKX
- Huobi
- Gate.io
- Gemini
- Bitstamp

**BRC20 and Runes**
- Ordiscan 
- CoinGecko
- CryptoCompare

**Stock Indexes**
- Financial Modeling Prep
- More coming soon ! 

Every 15 mins, the relayer in each Torram node sends the same information as merkle proofs to the Bitcoin chain using Bitcoin Core.  

## Prerequisites  

This guide provides step-by-step instructions for setting up the Torram node using a Docker image.

### Hardware 

- A dedicated machine or cloud server
- Minimum system requirements:
  - CPU: 2+ cores
  - RAM: 24 GB or more
  - Disk: 500GB SSD (or more for blockchain storage)
- Network: stable internet connection with at least 1 Mbps download/upload speed
- Operating System: Linux (Ubuntu preferred), macOS, or Windows Subsystem for Linux (WSL)

### Docker

Ensure you have Docker installed. Refer to [Docker installation](https://docs.docker.com/engine/install/)

### Bitcoin Core

Ensure you have [Bitcoin Core](https://bitcoincore.org/en/download/) downloaded and configured to **testnet3**. 

The Torram team can help you with testnet3 Bitcoin to get you started. Here are some additional faucets:

- https://coinfaucet.eu/en/btc-testnet/
- https://tbtc.bitaps.com/
- https://testnet.help/en/btcfaucet/testnet
- https://en.bitcoin.it/wiki/Testnet#Faucets
- https://testnet-faucet.mempool.co
- https://bitcoinfaucet.uo1.net

Your Bitcoin Core testnet3 wallet will need to have a balance always, as the Torram node will send transactions to the Bitcoin network every 15 minutes. 

### Software

As a Validator, you will be running **three** set of software.

1. Torram chain core
2. Bitcoin Core with testnet3
3. Torram chain's relayer 

## Setting Up Torram Chain Core

### 1. Genesis Validator Setup

First, initialize and start the genesis validator:

#### Load Docker Image

```bash
# For Windows
docker load < torramchainamd64.tar 

# For Mac Arm64
docker load < torramchainarm64.tar 

# For Ubuntu
docker load < torramchainamd64.tar
```

#### Run the Container

```bash
docker run -d \
  --name torram-validator \
  -p 4001:4001 \
  -p 1317:1317 \
  -p 9090:9090 \
  --network=host \
  torramchain:latest /usr/local/bin/validator
```

#### Initialize the Torram Chain

```bash
# Initialize the chain
docker exec -it torram-validator sh ./start.sh

docker exec -it torram-validator sh
```

#### Fetch Genesis Information

```bash
# Fetch genesis from the genesis validator
curl 34.57.91.248:26657/genesis | jq '.result.genesis' > ~/.torramd/config/genesis.json
```

### 2. Configure P2P Connection

```bash
# Get validator ID and configure peers
VALIDATOR_ID=$(curl -s http://34.57.91.248:26657/status | jq -r '.result.node_info.id')
CONFIG_DIR="$HOME/.torramd/config"
sed -i.bak'' "s/^seeds =.*/seeds = \"$VALIDATOR_ID@34.57.91.248:26656\"/" "$CONFIG_DIR/config.toml"
sed -i.bak'' "s/^persistent_peers =.*/persistent_peers = \"$VALIDATOR_ID@34.57.91.248:26656\"/" "$CONFIG_DIR/config.toml"
```

### 3. Configure State Sync

```bash
sed -i.bak'' 's|enable = true|enable = false|' "$CONFIG_DIR/config.toml"
sed -i.bak'' 's|^rpc_servers =.*|rpc_servers = "http://34.57.91.248:26657,http://34.57.91.248:26657"|' "$CONFIG_DIR/config.toml"
TRUST_HASH=$(curl -s "http://34.57.91.248:26657/block?height=$1" | jq -r .result.block_id.hash)
sed -i.bak -e "s/trust_height = 0/trust_height = 1/" $CONFIG_DIR/config.toml
sed -i.bak -e "s/trust_hash = \"\"/trust_hash = \"$TRUST_HASH\"/" $CONFIG_DIR/config.toml
```

### 4. List Keys

```bash
torramd keys list --keyring-backend test
```

### <span style="color:red;">🔴 IMPORTANT!</span>

Curently, the staking feature is an IOU. At this step, please provide the above generated address to the Torram team. The Torram team will send you IOU funds to complete the setup process. 

**Please don't move forward without completing this step with the Torram team** 

### 5. Verify Your Balance

Once you've provided your address to the Torram team and they've sent you funds, verify your balance:

```bash
torramd query bank balances <YOUR_ADDRESS> --node tcp://34.57.91.248:26657
```

Replace `<YOUR_ADDRESS>` with your actual address. This command confirms that you have received the necessary funds to proceed. The verification process can take anywhere from 1 minute to 5 minutes to complete.

### 6. Create New Validator

Once you've confirmed your balance, follow these steps to create your validator:

#### 6.1 Create Validator Configuration

### <span style="color:red; font-weight:bold;">🔴 Please add your own name for moniker ex torram-node 🔴</span>

```bash
# Get validator pubkey
PUBKEY=$(torramd tendermint show-validator)

cat > validator.json << EOF
{
    "pubkey": $(echo $PUBKEY),
    "amount": "1000000torram",
    "moniker": <ADD YOUR MONIKER NAME HERE>,
    "commission-rate": "0.1",
    "commission-max-rate": "0.2",
    "commission-max-change-rate": "0.01",
    "min-self-delegation": "1"
}
EOF
```

#### 6.2 Submit Create-Validator Transaction

```bash
torramd tx staking create-validator validator.json \
  --from=torram-node \
  --chain-id=torram \
  --gas="auto" \
  --gas-adjustment="1.5" \
  --fees=30000torram \
  --keyring-backend=test \
  --node tcp://34.57.91.248:26657
```
After running this command, you will be prompted confirm transaction before signing and broadcasting [y/N]:
message. Type "y" to confirm and proceed.


#### 6.3 Start the Validator Node

**Wait 2 mins before starting** 

```bash
torramd start
```

## Running the Bitcoin Core

### Step 1: Verify Bitcoin Testnet is Running
Before running the relayer, ensure that **Bitcoin Testnet 3** is active and fully synced:

```bash
bitcoin-cli -testnet getblockchaininfo
```

Check the `"initialblockdownload"` status. If it's `false`, your node is synced.

### Step 2: Check Bitcoin Configuration File
Verify the configuration file to ensure your Bitcoin node is set up correctly:

For ubuntu or windows
```bash
cat ~/.bitcoin/bitcoin.conf
```

For apple Sillicon 
```bash
cat ~/Library/Application\ Support/Bitcoin/bitcoin.conf
```


If running inside Docker:

```bash
docker exec -it bitcoin-node cat /root/.bitcoin/bitcoin.conf
```

*Sample bitcoin.config*

```
server=1
txindex=1
daemon=1
dbcache=2048
maxmempool=512
maxconnections=25

# Testnet-specific settings
[test]
testnet=1
rest=1
rpcuser=your_username
rpcpassword=your_secure_password
rpcallowip=0.0.0.0
rpcport=18332
rpcbind=0.0.0.0

# DNS Seeds
dnsseed=1
dns=1
```

### Step 3: Create & Load a Wallet
If you don’t have a wallet, create one:

```bash
bitcoin-cli -testnet createwallet "torram_wallet" false false "" false false false
```

If you already have the wallet, load it:

```bash
bitcoin-cli -testnet loadwallet "torram_wallet"
```

### Step 4: Generate a New Bitcoin Address
Create a new Bech32 (SegWit) address for receiving **testnet3 BTC**:

```bash
ADDRESS=$(bitcoin-cli -testnet -rpcwallet=torram_wallet getnewaddress "mylabel" "bech32")
echo "Your new address: $ADDRESS"
```

Send testnet3 BTC to this address using a faucet or ask the Torram team. 

### Step 5: Retrieve Your Private Key
Once you have received testnet3 BTC, retrieve the private key for the address:

```bash
bitcoin-cli -testnet -rpcwallet=torram_wallet dumpprivkey $ADDRESS
```

🔹 **Save this WIF key securely**—you’ll need it to run the relayer.

### Step 6: Check Your Wallet Balance
Ensure that testnet3 BTC has been received:

```bash
bitcoin-cli -testnet -rpcwallet=torram_wallet getbalance
```

### Step 7: Verify the Latest Bitcoin Block
Check the latest Bitcoin Testnet block:

```bash
bitcoin-cli -testnet getblockcount
bitcoin-cli -testnet getbestblockhash
```

You can also retrieve detailed block info:

```bash
bitcoin-cli -testnet getblock $(bitcoin-cli -testnet getbestblockhash)
```

## Running the Torram Relayer

Before running the relayer, if you need additional configuration details, check your Bitcoin configuration file:

For ubuntu or windows
```bash
cat ~/.bitcoin/bitcoin.conf
```

For apple Sillicon 
```bash
cat ~/Library/Application\ Support/Bitcoin/bitcoin.conf
```

If running inside Docker:

```bash
docker exec -it bitcoin-node cat /root/.bitcoin/bitcoin.conf
```

Once Bitcoin testnet3 is fully synced and your wallet is funded, start the **Torram Relayer**:

```bash
docker exec -it torram-validator relayer \
    -wif <your_wif_key> \
    -wallet torram_wallet \
    -rpcuser <your_rpc_username> \
    -rpcpass <your_rpc_password> \
    -rpchost localhost \
    -rpcport 18332
```

🔹 **Replace the placeholders (`<your_wif_key>`, `<your_rpc_username>`, `<your_rpc_password>`) with your actual values.**



## Monitoring

You can monitor the status of your validators using the following commands:

```bash
docker exec -it torram-validator sh

## Monitoring

You can monitor the status of your validators using the following commands:

```bash
docker exec -it torram-validator sh

# Query validator set
torramd query staking validators --node tcp://34.57.91.248:26657

# Query specific validator
torramd query staking validator <validator-address> --node tcp://34.57.91.248:26657
```

## Relayer Operation Flow

The relayer performs several key functions in sequence:

1. **Price Data Collection**
   ```
   [Price Fetcher] Waiting for price data...
   Starting price server on :8081
   Received data: {"height":62,"prices":{"BTC":"105119.135000000000000000","ETH":"3271.015000000000000000"}}
   ```

2. **Data Processing and Merkle Tree Generation**
   ```
   Processing batch of 10 items
   Extracted height: 44
   Calculated merkle root: f66be90e8e3631c924beda7657cb77799729b5fc9f45c94e62fba2aa5ae2dac9
   ```

3. **Bitcoin Transaction Submission**
   ```
   Transaction successfully broadcast! TxID: e05c4426c8e9dcec2bb7ff0ecab4a758b637cff987eeb42fa3da652f19896f96
   ```

4. **Confirmation Monitoring**
   The relayer continuously monitors for Bitcoin transaction confirmation:
   ```
   Waiting for confirmation of transaction: e05c4426c8e9dcec2bb7ff0ecab4a758b637cff987eeb42fa3da652f19896f96
   Raw API Response: {"confirmed":false}
   Waiting for confirmation... Transaction not yet confirmed
   ```

5. **Transaction Confirmation and Data Update**
   Once confirmed, the relayer updates both chains:
   ```
   Transaction confirmed! TxID: e05c4426c8e9dcec2bb7ff0ecab4a758b637cff987eeb42fa3da652f19896f96 at block height 67636
   Successfully submitted and confirmed batch with txHash: e05c4426c8e9dcec2bb7ff0ecab4a758b637cff987eeb42fa3da652f19896f96
   ```

## Data Storage and Verification

The relayer maintains a local database (oracle.txt) with entries like:
```json
{
    "height": 44,
    "prices": {
        "BTC": "105164.795000000000000000",
        "ETH": "3271.620000000000000000"
    },
    "processed": true,
    "tx_hash": "50b2c004a91da8ef2104d91b9fd6e88cba49a4afb88ae271e13774b4607cd27c",
    "cmd_tx_hash": "9308B24B2E78CB643493534CB164A9064D67F5DE2E83AFB9C8C78CF58D104B98"
}
```

## Transaction Verification

You can verify transactions on both chains:

1. **Bitcoin Testnet**
   Visit the transaction in the Bitcoin testnet explorer:
   https://mempool.space/testnet4/tx/50b2c004a91da8ef2104d91b9fd6e88cba49a4afb88ae271e13774b4607cd27c

2. **Torram Chain**
   Query the transaction details using the CLI:
   ```bash
   torramd query tx 9308B24B2E78CB643493534CB164A9064D67F5DE2E83AFB9C8C78CF58D104B98 --output json | jq
   ```