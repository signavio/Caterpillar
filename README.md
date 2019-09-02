# Caterpillar
BPMN execution engine on Ethereum

Caterpillar is a Business Process Management System (BPMS) that runs on top of Ethereum and that relies on the translation of process models into smart contracts. More specifically, Caterpillar accepts as input a process model specified in BPMN and generates a set of smart contracts that captures the underlying behavior. The smart contracts, written in Ethereum's Solidity language, can then be compiled and deployed to the public or any other private Ethereum network using standard tools. Moreover, Caterpillar exhibits a REST API that can be used to interact with running instances of the deployed process models.

Caterpillar also provides a set of modelling tools and an execution panel which interact with the underlying execution engine via the aforementioned REST API. The latter can also be used by third party software to interact in a programmatic way via Caterpillar with the instances of business process running on the blockchain.

The prototype has two versions. We are currently developing the version v2.0, which follows a different approach from v1.0. However, we kept the source code of v1.0 as a reference, although we are not currently working on such an implementation.

The approach implemented by the v2.0 can be accessed from: https://arxiv.org/abs/1808.03517.

The paper describing the Role Dynamic Binding and Access Control implemented by v2.1 can be accessed from: https://arxiv.org/abs/1812.02909.

More detailed documentation on how to use Caterpillar (v2.0 and v2.1) can be found at:  https://github.com/orlenyslp/Caterpillar/blob/master/v2.1/CaterpillarDoc.pdf

Additionally, a demo paper about v1.0: can be accessed from: http://ceur-ws.org/Vol-1920/BPM_2017_paper_199.pdf

Caterpillar’s code distribution in this repository contains three different folders in v1.0 and two in v2.0. The folder __caterpillar_core__ includes the implementation of the core components, __execution_panel__ consists of the code of a BPMN visualizer that serves to keep track of the execution state of process instances and to lets users check in process data. The __services_manager__ folder contains the implementation for an external service which is used only in v1.0 for demonstration purposes. In v2.1, the source code is in the folder labelled as "prototype". Besides, the folder "Dynamic Binding Example" contains the binding policy and BPMN model used as running example in https://arxiv.org/abs/1812.02909.

For running Caterpillar locally, download the source code from the repository and follow the next steps to set up the applications and install the required dependencies. For running caterpillar from a Docker image go directly to the last section of this document. Be aware that the Docker image works only on the version v1.0.

## Setting the hyperledger network

In this step, we will setup fabric locally using docker containers, install the EVM chaincode (EVMCC) and instantiate the EVMCC on our fabric peers. This step uses the hyperledger [fabric-sample](https://github.com/hyperledger/fabric-samples) repo to deploy fabric locally and the [fabric-chaincode-evm](https://github.com/hyperledger/fabric-chaincode-evm) repo for the EVMCC, Fab3 and Fab3.  This step follows the [fabric-chaincode-evm tutorial](https://github.com/hyperledger/fabric-chaincode-evm/blob/master/examples/EVM_Smart_Contracts.md) closely.

### Clean docker and set GOPATH

This will remove all your docker containers and images!
```
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
docker rmi $(docker images -q) -f
```

Make sure to set GOPATH to your go installation
```
export GOPATH=$HOME/go
```

### Get Fabric Samples and download Fabric images

Clone the [fabric-samples](https://github.com/hyperledger/fabric-samples) repo in your `GOPATH/src/github.com/hyperledger` directory:
```
cd $GOPATH/src/github.com/hyperledger/
git clone https://github.com/hyperledger/fabric-samples.git
```

Checkout `release-1.4`
```
cd fabric-samples
git checkout release-1.4
```

Download the docker images
```
./scripts/bootstrap.sh
```

### Mount the EVM Chaincode and start the network

Clone the `fabric-chaincode-evm` repo in your GOPATH directory.
```
cd $GOPATH/src/github.com/hyperledger/
git clone https://github.com/hyperledger/fabric-chaincode-evm
```

Checkout `release-0.1`
```
cd fabric-chaincode-evm
git checkout release-0.1
```


Now navigate back to your fabric-samples folder.  Here we will use first-network to launch the network.
```
cd $GOPATH/src/github.com/hyperledger/fabric-samples/first-network
```

Update the `docker-compose-cli.yaml` with the volumes to include the fabric-chaincode-evm.

```
  cli:
    volumes:
      - ./../../fabric-chaincode-evm:/opt/gopath/src/github.com/hyperledger/fabric-chaincode-evm
```

Generate certificates and bring up the network
```
./byfn.sh generate
./byfn.sh up
```

### Install and Instantiate EVM Chaincode

Navigate into the cli docker container
```
docker exec -it cli bash
```

If successful, you should see the following prompt
```
root@0d78bb69300d:/opt/gopath/src/github.com/hyperledger/fabric/peer#
```

To change which peer is targeted change the following environment variables:
```
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
```

Next install the EVM chaincode on all the peers
```
peer chaincode install -n evmcc -l golang -v 0 -p github.com/hyperledger/fabric-chaincode-evm/evmcc
```

Instantiate the chaincode:

```
peer chaincode instantiate -n evmcc -v 0 -C mychannel -c '{"Args":[]}' -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

Great.  You can exit out of the cli container and return to your terminal.
```
exit
```

You are now ready to setup Fab3.


### Setup Fab3


Execute the following to set certain environment variables required for setting up Fab3.

```
export FABPROXY_CONFIG=${GOPATH}/src/github.com/hyperledger/fabric-chaincode-evm/examples/first-network-sdk-config.yaml # Path to a compatible Fabric SDK Go config file
export FABPROXY_USER=User1 # User identity being used for the proxy (Matches the users names in the crypto-config directory specified in the config)
export FABPROXY_ORG=Org1  # Organization of the specified user
export FABPROXY_CHANNEL=mychannel # Channel to be used for the transactions
export FABPROXY_CCID=evmcc # ID of the EVM Chaincode deployed in your fabric network
export PORT=8545 # Port the proxy will listen on. If not provided default is 8545.
```

Navigate to the `fabric-chaincode-evm` cloned repo:
```
cd $GOPATH/src/github.com/hyperledger/fabric-chaincode-evm/
```
Run the following to build the fab proxy
```
go build -o fab3 ./fabproxy/cmd
```
You can then run the proxy:
```
./fab3
```

This will start Fab3 at `http://localhost:8545`

You should see output like the following:
```
{"level":"info","ts":1550530404.3546276,"logger":"fab3","caller":"cmd/main.go:143","msg":"Starting Fab3","port":8545}
```

## How to use Caterpillar Core

> Before running Caterpillar Core, make sure you installed gulp-cli running the command: npm __install gulp-cli -g__. All the instructions about the glup-cli installation can be found here: https://www.npmjs.com/package/gulp-cli?activeTab=readme

> Additionally, the version v2.0 uses a process repository to store and access metadata produced by Caterpillar when compiling the BPMN model into Solidity smart contracts. Currently, this repository was implemented on top of MongoDB which is a database that stores data as JSON-like documents. The instructions to install MongoDB Community Edition can be accessed from here: https://docs.mongodb.com/manual/administration/install-community/

To set up and run the core, open a terminal in your computer and move into the folder __caterpillar_core__.

For installing the dependencies, run the commands

     npm install
     gulp build

For running the application you may use one of the following commands

     node ./out/www.js
     gulp

By default the application runs on http://localhost:3000.

> Make sure you have fabric running in your computer before starting the core. Besides, if you are using the version v2.0 a MongoDB client must be also running.

The application provides a REST API to interact with the core of Caterpillar. The following table summarizes the mapping of resource-related actions:

| Verb | URI                       | Description                                                            |
| -----| ------------------------- | ---------------------------------------------------------------------- |
| POST | /models                   | Registers a BPMN model (Triggers also code generation and compilation) |
| GET  | /models                   | Retrieves the list of registered BPMN models                           |
| GET  | /models/:mid              | Retrieves a BPMN model and its compilation artifacts                   |
| POST | /models/:mid              | Creates a new process instance from a given model                      |
| GET  | /processes/               | Retrieves the list of active process instances                         |
| GET  | /processes/:pid           | Retrieves the current state of a process instance                      |
| POST | /workitems/:wimid/:wiid   | Checks-in a work item (i.e. user task)                                 |
| POST | /workitems/:wimid/:evname | Forwards message event, delivered only if the event is enabled         | 

> (ONLY FOR v1.0) If the process model has service tasks and consequently needs to interact with the services manager application, then you must run the services manager application and create the corresponding services before running Caterpillar core.

From the caterpillar_core folder, it is possible to run the script:

     node demo_running_example_test.js

Which is provided only for the version v1.0, to register, create an instance and get the address of a sample process provided in the file __demo_running_example.bpmn__. For running the sample process, it is also required to run first the application __services_manager__ and to register the external services.

### Updates in v2.1

The version v2.1 extends v2.0 with an Access Control mechanism, based on a role dynamic binding policy. Accordingly, the REST API to interact with the core of Caterpillar was extended with the following resource-related actions.


| Verb | URI                       | Description                                                                         |
| -----| ------------------------- | ----------------------------------------------------------------------------------- |
| POST | /registry                 | Deploys a new instance of the Runtime Registry                                      |
| POST | /resources/policy         | Generates/Deploys the contracts from a given binding policy specification (BPF)     |
| POST | /resources/task-role      | Generates/Deploys the contract mapping the roles to the tasks in a given BPMN model |
| POST | /resources/nominate       | Nominates an actor into a role as defined in the BPF                                |
| POST | /resources/release        | Releases an actor from a role as defined in the BPF                                 |
| POST | /resources/vote           | Accepts/Rejects the nomination/release of an actor as defined in the BPF            |
| GET  | /resources/:role/:pAddr   | Retrieves the current state of an actor into a role in a process instance           |

The grammar describing the Binding Policy Specification can be found at: https://github.com/orlenyslp/Caterpillar/blob/master/v2.1/prototype/caterpillar-core/src/models/dynamic_binding/antlr/binding_grammar.g4.

The full description of the language and Dynamic Role Binding can be found at: https://arxiv.org/abs/1812.02909 


## How to use Services Manager (ONLY FOR v1.0)

Open a terminal in your computer and move into the folder __services_manager__. 

For installing the dependencies, run the comand 

     npm install
     gulp build

For running the application use the comand 

     node ./out/www.js

By default the application will runs on http://localhost:3010.

The application provides a REST API to interact with services manager. The following table summarizes the mapping of resource-related actions:

| Verb | URI            | Description                                                            |
| -----| -------------- | ---------------------------------------------------------------------- |
| POST | /services      | Registers an external service                                          |
| GET  | /services      | Retrieves the list of registered external services                     |
| GET  | /services/:sid | Retrieves smart contract/address of an external service                |

From the __services_manager__ folder is possible to run the script

     node oracle_creation.js

to register the services required by the running example provided in the core.

## How to use Execution Panel

> Before running the Execution Panel, make sure you installed angular-cli: https://github.com/angular/angular-cli/wiki

To set up and run the execution panel, open a terminal in your computer and move into the folder __execution-panel__.

For installing the dependencies, run the comand

     npm install

For running the application use the comand

     ng serve

Open a web browser and put the URL http://localhost:4200/.

You must use the button refresh to update the instances running, and then select one of the URLs obtained that will contain the address when it is running the smart contract. Then press the button __Open__. Here, you can see the enabled activities visualized in dark green. For executing any enabled activity, just click on it and fill the parameter info if required. All the execution of the process (including internal operations) can be traced from the terminal of __caterpillar_core__.

## Deploying the smart contract into hyperledger fabric

Next, we'll install the web3 dependency and use this library to deploy the smart contract.

To install the same version of `web3` open a new terminal and run:
```npm install web3@0.20.2 ```

Now we'll enter node console to set up our web3.

```node```

Assign Web3 library and use the fab3 running in the previous terminal as provider.
```
Web3 = require('web3')
web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:8545'))
```
Check to see your account information
```
web3.eth.accounts
```
You should see something like this which is similar to an Ethereum account:
  ```
  [ '0x2c045d4565e31cef1f6cd7368c3436a79f1cea4f' ]
  ```
Assign this account as defaultAccount
```
web3.eth.defaultAccount = web3.eth.accounts[0]
```

* The terminal which has the caterpillar core opened returns the byteCode value and the interface value which is the ABI. Assign the interface value to ABI.

``` ABI = interface ```
* Next assign the long evm complied byte code:  

ByteCode = `608060405234801561001057600080fd5b506..........9899`


Assign contract with web3 using the contract's ABI.
```
Contract = web3.eth.contract(ABI)
```

Next, deploy the contract using the contract byte code.
```
deployedContract = Contract.new([], { data: ByteCode })
```

You can get the contract address by using the transaction hash of the deployed contract.
```
web3.eth.getTransactionReceipt(deployedContract.transactionHash)
```

The smart contract is deployed to the hyperledger network.


