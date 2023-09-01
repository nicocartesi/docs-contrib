---
layout: post
title:  "How to Create Your First Cartesi DApp"
date:   2023-08-30
author: "Eric Mboizi"
---

## How to Create Your First Cartesi DApp by Eric Mboizi

Cartesi Rollups is an Application Specific Rollups solution with a Linux runtime. It enables you to create verifiable, repeatable, & transparent computations. We can visualize Cartesi in two ways: the blockchain (on-chain) and node (off-chain) perspectives. 

The off-chain perspective encompasses the Cartesi Machine and the client application. The Cartesi Machine is a RISC-V emulator running on the Linux OS. On the other hand, the on-chain perspective has the Rollups smart contracts and the DApp smart contracts. 

In this tutorial, we show how these two perspectives come together. We will do this by building an ASCII canvas (a drawing board) using Python. The blockchain will be our canvas. 

The aim is for each user (Ethereum address) to be able to add their ASCII-drawn name to the Cartesi DApp.

Let’s get started!

### Pre-requisites 

First, let’s list all the tools that we’ll need to create and run our application:

1. Python >= 3.1

2. Python library for drawing ASCII art. You can install it using the command below:
   ```python
   python3  -m pip install art
   ```

3. Web3.py Library. You can install it using:

   ```python
   python3 -m pip install web3
   ```

4. Docker. In case you don't have it installed, you can refer to [this documentation](https://docs.docker.com/engine/install/) for your particular operating system.

### Overview 

As mentioned earlier, the application is a simple project that turns given input into ASCII art. Here is a sample output of the ASCII art on the terminal:

![Overview]({{ site.baseurl }}/images/2023-08-30-how-to-create-your-first-cartesi-dapp/image1.png)

We can look at our application in terms of having the frontend and the backend. The frontend is what we'll use to send our data (i.e. a name) to the *deployed* **InputBox** contract. 

The node listens for changes on the blockchain and then notifies our application's backend of them. After executing the DApp logic, we can then post the results (outputs) to the blockchain.

Our backend runs inside a Cartesi Node; it is our smart contract and acts like so. The node is embedded with the Cartesi Machine, where the backend logic actually runs.

### Setting Up the Development Environment 

First, clone this repository:

```shell
git clone https://github.com/Mberic/ascii-python.git
```

Let's explain the files you have just downloaded. 

1. **docker-compose.yml** – This file contains the Docker images that make up a Cartesi node. A node contains several components including, the State-fold server, server manager, PostgreSQL database, etc. If you would like to know about the architecture of the node components, you can refer to [this documentation](https://github.com/cartesi/rollups/tree/main/offchain#architecture-overview). 

2. **docker-compose-host.yml** – This sets up our server manager for use in host mode.

3. **frontend.py** – This file sends a name to the **InputBox** contract.

4. **backend.py** – This file processes the requests and sends back responses. For our application, the aim is simply to register that the backend received the name. 
   Therefore, we'll advance the state of our DApp by sending  a notice. 

Now, let's run the development environment in host mode: 

```shell
docker compose -f ./docker-compose.yml -f ./docker-compose-host.yml up
```

Host mode means that you can run the backend on your local machine just like any other applications you normally do (instead of running it in the Cartesi node). This is helpful for purposes such as debugging and of course for experimentation, like we're doing right now.

Running the above command will start a local hardhat environment and also instantiate a Cartesi node. It will start up a number of services at different ports, like: 4000, 5004, 8545. You can check the Docker files to understand which services these ports correspond to.

You need to look at the output on the terminal to get the contract addresses that we’ll be using in the **frontend.py** file. Two particular contracts are of interest to us:

1. **InputBox**: 0x5a723220579C0DCb8C9253E6b4c62e572E379945
2. **CartesiDApp**: 0x142105FC8dA71191b3a13C738Ba0cF4BC33325e2

After some time has passed, you should see continuous block logs as below:

![Blocks]({{ site.baseurl }}/images/2023-08-30-how-to-create-your-first-cartesi-dapp/image2.png)

Each Cartesi DApp has a deployed **CartesiDApp** contract. The address of this contract is required when sending data to the Cartesi Rollups smart contracts as we will see later on. This address associates a given data input with a specific Cartesi DApp.

**NOTE**: Hardhat doesn't keep track of the previous changes you made while running it. Therefore, you need to check the contract addresses whenever you start the development environment. 

### Frontend: Interacting with the InputBox Contract

Among the key design features of Cartesi is that the computations are reproducible. If two nodes were to disagree with the input data, there would be no way to resolve this conflict. To ensure data consistency, all Cartesi DApps need to submit the data to advance their state through the Cartesi Rollups contracts. 

To do this, we need to interact with the deployed **InputBox** contract. This contract accepts arbitrary data to a corresponding **CartesiDApp** contract. It has the **addInput** function which takes in 2 inputs:  (1) the address of the **CartesiDApp** & (2) the data to be sent:

```solidity
   function addInput(
       address _dapp,
       bytes calldata _input
   ) external override returns (bytes32)
```

Now, open **a new terminal window** and run the **frontend.py** file. This will call the **addInput** method to send the ASCII text (name) to our CartesiDApp contract.

Note that the ASCII name we're sending doesn't carry the formatting that the Python **art** library uses in **frontend.py**. We just want to keep things simple. We assume that the sent text (name) uses the default formatting of the library. 

At our backend, the **State-Fold** server ( a component of the Cartesi node ) listens for state changes in the blockchain and reports them to our offchain machine. We can then carry out some processing and submit the results back to the blockchain. 

### Backend: Advancing the Application's State

Now that our “frontend” has successfully added its input to the blockchain, we can now get the data from the blockchain and process it. This is where the Cartesi Machine comes in - it is our execution layer. 

Since we are running in host mode, our backend will be processed on our local environment. 

In a **new terminal window**, let's run the **backend.py** file in a Python virtual environment:

```shell
python3 -m venv .env
. .env/bin/activate
pip install -r requirements.txt
ROLLUP_HTTP_SERVER_URL="http://127.0.0.1:5004" python3 backend.py
```

You should see logs indicating that the application is waiting for requests to advance its state. When the name reaches the backend, details of the transaction will be displayed. Look at the image below for example.

![Log]({{ site.baseurl }}/images/2023-08-30-how-to-create-your-first-cartesi-dapp/image3.png)


The payload carries the name. Changing the hex payload above ( **0x4a656c6c7966697368** ) to ASCII will give you the name **Jellyfish**. You can play around with more names and observe the output.

If you are done working with the Rollups SDK environment, you can gracefully stop the Docker containers with:

```shell
docker compose -f ./docker-compose.yml -f ./docker-compose-host.yml down -v
```

### Summing It Up

In this tutorial, you have seen how to build your Cartesi application with the Cartesi Rollups SDK. Now, it’s your turn to implement your own DApp. 

We can’t wait to see what you BUILD!
