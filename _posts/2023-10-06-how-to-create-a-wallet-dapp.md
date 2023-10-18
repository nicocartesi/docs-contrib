# How to Build a Wallet DApp 
Cartesi enables you to transfer assets from the base layer to the execution layer. You can achieve this using a set of smart contracts called Portals. 

In this guide, we'll demonstrate what you can do with this infrastructure by building a basic wallet application in Python. 

The wallet supports simple operations such as announcing new deposits from the underlying chain and keeping track of account balances.

Let's get started!

## Setting Up
Installing Dependencies 
Here are the tools we'll need to create and run our application:

1. Python >= 3.1
2. Docker. In case you don't have it installed, you can refer to [this documentation](https://docs.docker.com/engine/install/) for your OS.
3. Npm >= v7.1
4. You'll also need knowledge of how to build, test, and deploy your smart contracts using Hardhat. In case you're unfamiliar with it, you can refer to [this introduction](https://hardhat.org/tutorial) on their site.

### Starting the Development Environment

First, download the files from this repository: 

```
git clone https://github.com/Mberic/wallet-dapp.git

```
Let's explain the files you've just downloaded:

- The **frontend** directory contains the frontend files of our wallet DApp. 

- The **backend** directory contains the Docker files which instantiate a Cartesi node and a local Ethereum network using Hardhat. It also has the backend logic of our DApp.

- The **tokens** directory includes the smart contracts for the ERC20,  ERC721, and ERC1155 tokens that we'll deploy to our local Hardhat network. The contracts are conveniently named to correspond to the tokens mentioned.     
These contracts mint the tokens to one of the local Hardhat addresses (0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266).
Please note that these contracts are arbitrary. There's nothing particularly special about them. Therefore, you can create your own and deploy using one of the Hardhat addresses. The goal is to simply have some tokens on the base layer that we can then teleport to the above layer.

Now, let's start the Hardhat network and Cartesi node using the Docker files earlier mentioned:

```
cd backend/node
docker compose -f ../docker-compose.yml -f ./docker-compose-host.yml up
```

Look at the command line output. 

![Node-Initialisation]({{ site.baseurl }}/images/2023-10-06-how-to-create-a-wallet-dapp/image1.png)

It displays the addresses of the **Portal** contracts deployed on the local testnet. We'll need the following addresses later on:

- **EtherPortal:** 0xFfdbe43d4c855BF7e0f105c400A50857f53AB044
- **ERC20Portal:** 0x9C21AEb2093C32DDbC53eEF24B873BDCd1aDa1DB
- **ERC721Portal:** 0x237F8DD094C0e47f4236f12b4Fa01d6Dae89fb87
- **ERC1155SinglePortal:** 0x7CFB0193Ca87eB6e48056885E026552c3A941FC4
- **CartesiDapp:** 0x70ac08179605AF2D9e75782b8DEcDD3c22aA4D0C

The output will also show 20 Hardhat addresses and their private keys.

Next, let's deploy the ERC20, ERC721, and ERC1155 token contracts to the Hardhat network.  In a new terminal window, navigate to the **token** directory and run:
```
npm install hardhat
npx hardhat compile
npx hardhat run scripts/deploy.js --network localhost
```
You can take note of the deployed addresses and [add Hardhat to Metamask](https://support.metamask.io/hc/en-us/articles/360043227612-How-to-add-a-custom-network-RPC#:~:text=Adding%20a%20network%20manually&text=MetaMask%20will%20open%20in%20a,Save'%20to%20add%20the%20network.). 

![Add-Hardhat-to-Metamask]({{ site.baseurl }}/images/2023-10-06-how-to-create-a-wallet-dapp/image2.png)

Afterward, you can [add the tokens to Metamask](https://support.metamask.io/hc/en-us/articles/360015489031-How-to-display-tokens-in-MetaMask#h_01FWH492CHY60HWPC28RW0872H) and change the network to Hardhat to verify the presence of the tokens on your Hardhat wallet address (0xf39F...266). This is actually the first address from the logs when initializing the node, and it’s the one you’ll add to Metamask as your account for this tutorial.

After setting up all this, it's now time to teleport the assets to our wallet application. 

## Portals

As mentioned earlier, portals allow us to transfer (teleport) assets from the base layer to the Cartesi network. Cartesi currently supports the transfer of the following types of assets: Ether, ERC20, ERC721 and ERC1155. 

Portals transfer the assets to our **CartesiDApp** contract. Additionally, each Portal contract sends an input for the associated CartesiDApp contract. You can check the input encoding for the supported token standards [from here](https://github.com/cartesi/rollups-contracts#input-encodings-for-deposits).

Let's look at how we can send these tokens to our wallet DApp.
EtherPortal

This contract is used for teleporting Ether. You can call the function *depositEther( address _dapp, bytes calldata _execLayerData)* from the deployed contract. 

We can then assign some representation to this token in the execution layer. 

For our example, we'll simply use a 1:1 correspondence. That is, 1 token in the base layer represents 1 execLayer token. As mentioned earlier, whenever you call a Portal contract, it sends some input to the DApp about the transaction. After getting this info, our offchain logic can then credit the user's address on the execution layer. 

Let’s run the **backend** script (located in **backend/node**). This file monitors new deposits and updates account balances accordingly.
```
python3 -m venv .env
. .env/bin/activate
pip install -r requirements.txt
ROLLUP_HTTP_SERVER_URL="http://127.0.0.1:5004" python3 backend.py
```
For example, here’s how script unpacks the ABI-encoded payload from the EtherPortal.

 ```py
       binary = bytes.fromhex(payload[2:])
       decoded = decode_packed(['address','uint256'],binary)
       global ether_balance
       ether_balance += decoded[1]
```

Now, navigate to the **frontend** directory & run the **deposit-ether** file in a new terminal window. This script sends ETH to the DApp contract. 
```
cd frontend
python3  deposit-ether.py
```
You can check the file to see how it uses the **depositEther** function:
```py
transaction = contract.functions.depositEther( CartesiDapp, hexData ).build_transaction( {
   'gasPrice': w3.eth.gas_price,
   'chainId': 31337,
   'from': HardhatWalletAddress,
   'value': 503450,
   'nonce': nonce, 
})
```

Take notice of the terminal window running the **backend** file. You'll see an output such as the one below (after sending Ether):

![Transaction-Info]({{ site.baseurl }}/images/2023-10-06-how-to-create-a-wallet-dapp/image3.png)


You should be able to see the decoded input from the transaction. This implies that you have successfully moved ETH from the base layer. With your ETH now at the execution layer, you can now send these tokens to another user. You can also check your Metamask wallet to verify that the tokens have actually been removed from your wallet.

Another functionality that our DApp is able to achieve is updating token balances. You can check these balances (based on token deposits) by running the **check-balances** file and then viewing the log output at the backend.

## ERC20, ERC721 & ERC1155 Portals 

Our wallet can also send transactions for ERC20, ERC721, and ERC1155 tokens ( through the corresponding frontend files). Their implementation is similar to that of **deposit-ether.py**. However, the [functions you need to call](https://docs.cartesi.io/cartesi-rollups/api/json-rpc/portals/) are different:

- ERC20Portal – depositERC20Tokens
- ERC721Portal – depositERC721Token
- ERC1155SinglePortal – depositSingleERC1155Token
- ERC1155BatchPortal – depositBatchERC1155Token

Note that you need to approve a spending allowance for ERC20, ERC721 & ERC1155 deposits. This means calling the corresponding **approve** method for each **deployed token contract** in order to allow the given Portal contract permission to transfer an asset from your account.

For example, you can have such an implementation for teleporting an ERC20 token:
```py
# approve allowance for ERC20Portal
token_transaction = token_contract.functions.approve(ERC20Portal, amount*5).build_transaction({
   'gasPrice': w3.eth.gas_price,
   'chainId': 31337,
   'from': HardhatWalletAddress,
   'nonce': nonce, 
})

# sign the transaction
signed_token_txn = w3.eth.account.sign_transaction(token_transaction, private_key = PrivateKey)

# send the transaction
w3.eth.send_raw_transaction(signed_token_txn.rawTransaction) 

# transfer an amount less than the approved value e.g amount < amount * 5
portal_transaction = portal_contract.functions.depositERC20Tokens( ERC20_Token_Address, CartesiDapp, amount, execLayerHexData ).build_transaction( {
   'gasPrice': w3.eth.gas_price,
   'chainId': 31337,
   'from': HardhatWalletAddress,
   'nonce': nonce + 17, 
})


signed_portal_txn = w3.eth.account.sign_transaction(portal_transaction, private_key = PrivateKey)
result = w3.eth.send_raw_transaction(signed_portal_txn.rawTransaction)
tx_hash = result.hex() # transaction hash

print("Transaction hash: " + tx_hash)
```
At the terminal running the backend, you’ll see the transaction info for your deposit after you’ve made an ERC20 deposit to the DApp.

## Running Inside the Cartesi Machine (Production Mode)

In the sections above, our backend has been running on the local OS and hence acting as our node. However, in a production setting, your backend logic needs to run inside the Cartesi Machine (a RISC-V Linux environment). Let's see how to achieve this.

First, stop the docker containers we had earlier started:

```
docker compose -f ./docker-compose.yml -f ./docker-compose-host.yml down -v
```

Next, ensure that your Docker environment supports the RISC-V platform. You should see **linux/riscv64** among the options listed after running the following command:

```
docker buildx ls
```

If you don't see it, then install a virtualization tool called qemu:

```
sudo apt install qemu-user-static
```

Afterward, run **docker buildx** ls to confirm RISC-V support. 

Next, navigate to the **node directory**. To run the backend inside the Cartesi node, enter the following command in your terminal:

```
docker buildx bake  --load
```

This will build a Cartesi node image with the DApp backend in it. To start the environment, run:

```
docker compose -f ../docker-compose.yml -f ./docker-compose.override.yml up
```

Take notice of the logs. They show output for both the Cartesi node and our DApp. Therefore, if you send an input with the frontend files, you’ll see the backend output from here.

After you’re done using the environment, you can shut it down using:
```
docker compose -f ../docker-compose.yml -f ./docker-compose.override.yml down -v
```