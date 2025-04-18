## DevOps docs
All workflows are limited to PRs with target=master for the sake of simplicity in this demo.
In the future, a feature should be implemented which allows to build images and deploy contracts from feature branches.

### Build
When a PR with label CI:Build is merged in the master branch, a Docker image is build and pushed tagged with latest AND with the commit ID. This way you can refer to older version if needed but at the same time latest is available.

!NB If you encounter a problem with pushing to the registry, check if the access token has not expired. It has 30 days expiration.

### Deploy

#### Deploying the contract
Although hardhat ignition works perfectly with the hardhat network, there are issues when trying to deploy to Geth.

To solve this, there is deploy script in `hardhat/scripts` dir which deploys the sample Lock contract to the blockchain.

If you need to deploy the contract locally:
1. cd to `hardhat` dir
2. run `npx hardhat run scripts/deploy.ts --network geth`

#### Creating new image containing the contract
I've decided to go with mounted dir for the data dir because it is easier to create new dokcer image with the contract already deployed. If I've stayed on the volume mount, it would require more steps to persist the data dir.
Using mounted dir for the data can be tricky because of permission errs tho.

Currently, the new image (with contract) is built from the latest base image.
This should be extended in the future to support building from a given tag.

### Tests
The default tests that Hardhat Sample Project provides depends on using the Hardhat network.
This is an example of an error they throw when ran against the Geth:
```
  4) Lock
       Deployment
         Should fail if the unlockTime is not in the future:
     OnlyHardhatNetworkError: This helper can only be used with Hardhat Network. You are connected to 'geth'.
      at checkIfDevelopmentNetwork (<obfustcated>/hardhat/node_modules/@nomicfoundation/hardhat-network-helpers/src/utils.ts:30:11)
      at getHardhatProvider (<obfuscated>/hardhat/node_modules/@nomicfoundation/hardhat-network-helpers/src/utils.ts:41:9)
      at async Object.latest (<obfuscated>/hardhat/node_modules/@nomicfoundation/hardhat-network-helpers/src/helpers/time/latest.ts:7:20)
      at async Context.<anonymous> (<obfuscated>/hardhat/test/Lock.ts:54:26)
```
I've provided the same tests reworked to be compatible with Geth.

When running the docker container with Geth, I do not use bind mount and I am making sure that the geth_data dir from previous steps is cleaned so I can be sure that I am working with the contract previously deployed in the container.

### Infrastructure
All Infrastructure as a Code is stored inside `infrastructure` dir. I went with GCP because of the free trial (AWS was exhausted).

Currently, the IaC is ment to be used locally. Later, a pipeline should be create and the state must be stored remotely and securely.

The `secrets` directory is holding all infrastrucutre related sensitive data. It is excluded from source control. 

To authenicate the provider, place your service account key JSON file in the `secrets` dir and name it `svc_acc_key.json`.
This is not a requirement and you can change this as you like, but this is the fastest and easiest way to get up and running.

Please name your service account `terraformer` when creating it in the GCP WEB UI for ease of use.

#### Modules

**gcp-apis**
This module is responsible for enabling all APIs that will be needed for the project.

Required inputs:
- `project_id`

!NB There is known issues with enabling APIs through Terraform. Sometimes the backend does not return a response quickly enough which results in the API being enabled really but terraform apply fails. Rerun the pipeline/command.
There are workaround but for the sake of this demo please use the solution described above.

**gcp-networking**
Holds code responsible for setting up all networking resource needed for a k8s cluster to operate - router and a NAT gateway.

Required inputs:
- `project_id`

**gcp-gke-cluster**
The quota for the free trial is being an issue for the demo. I can't raise the quota limit and I am using the min. disk size (10GB). 
To overcome this limitation I had to use zonal cluster with only 2 nodes.

Required inputs:
- `project_id`

#### How to
Prerequisite:
- Enable Service Usage API from GCP WEB UI
- Cloud Resource Manager API

1. Go to GCP WEB UI > IAM & Admin > Service Accounts;
2. Create new Service Account with name `terraformer` and assign Owner role (for production this should be limited role, never use owner);
3. Create new service account key and download it as JSON;
4. Put the key in `infrastructure/secrets` dir and name it `svc_acc_key.json`;
5. Open new terminal window and make sure you are in `infrastructure` dir;
6. Run `terraform init`;
7. Run `terraform plan` and revisit the plan output;
8. Run `terraform apply`. Wait for the prompt or just add `-auto-approve` to automatically approve the action;
9. Run `terraform destroy` after you are done with testing and messing around.

### Deploy to k8s
For the sake of simplicity I am giving steps to deploy the container to k8s from local machine.
However, if this was a real task there should be a pipeline which do taht.
Also, the pipe should contain logic to wait for the deployment to be up and running before proceeding with next steps for example.

k8s related files are located in `k8s` dir.

Prerequisite:
- Get gcloud cli and authenticate it;
- Install the follow gcloud component - gke-gcloud-auth-plugin; ( gcloud components install gke-gcloud-auth-plugin)
1. Authenticate against the test-cluster. If you are following along:
`gcloud container clusters get-credentials test-cluster --zone europe-west1-b`
2. cd to `k8s` directory;
3. Run `kubectl apply -f .` to deploy the app;
4. Use `kubectl port-forward` to access the deployment if needed;

### Blockscout
Everything is moved inside `docker-compose` dir.

I've reused the configs from the official repo but added geth with my image (latest).
When I search the address of the developer account that Geth creates by default I get a result.
If I search the contract address after deploying it again I am getting a result.

---

## Go Ethereum

Golang execution layer implementation of the Ethereum protocol.

[![API Reference](
https://pkg.go.dev/badge/github.com/ethereum/go-ethereum
)](https://pkg.go.dev/github.com/ethereum/go-ethereum?tab=doc)
[![Go Report Card](https://goreportcard.com/badge/github.com/ethereum/go-ethereum)](https://goreportcard.com/report/github.com/ethereum/go-ethereum)
[![Travis](https://app.travis-ci.com/ethereum/go-ethereum.svg?branch=master)](https://app.travis-ci.com/github/ethereum/go-ethereum)
[![Discord](https://img.shields.io/badge/discord-join%20chat-blue.svg)](https://discord.gg/nthXNEv)

Automated builds are available for stable releases and the unstable master branch. Binary
archives are published at https://geth.ethereum.org/downloads/.

## Building the source

For prerequisites and detailed build instructions please read the [Installation Instructions](https://geth.ethereum.org/docs/getting-started/installing-geth).

Building `geth` requires both a Go (version 1.23 or later) and a C compiler. You can install
them using your favourite package manager. Once the dependencies are installed, run

```shell
make geth
```

or, to build the full suite of utilities:

```shell
make all
```

## Executables

The go-ethereum project comes with several wrappers/executables found in the `cmd`
directory.

|  Command   | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| :--------: | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`geth`** | Our main Ethereum CLI client. It is the entry point into the Ethereum network (main-, test- or private net), capable of running as a full node (default), archive node (retaining all historical state) or a light node (retrieving data live). It can be used by other processes as a gateway into the Ethereum network via JSON RPC endpoints exposed on top of HTTP, WebSocket and/or IPC transports. `geth --help` and the [CLI page](https://geth.ethereum.org/docs/fundamentals/command-line-options) for command line options. |
|   `clef`   | Stand-alone signing tool, which can be used as a backend signer for `geth`.                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
|  `devp2p`  | Utilities to interact with nodes on the networking layer, without running a full blockchain.                                                                                                                                                                                                                                                                                                                                                                                                                                       |
|  `abigen`  | Source code generator to convert Ethereum contract definitions into easy-to-use, compile-time type-safe Go packages. It operates on plain [Ethereum contract ABIs](https://docs.soliditylang.org/en/develop/abi-spec.html) with expanded functionality if the contract bytecode is also available. However, it also accepts Solidity source files, making development much more streamlined. Please see our [Native DApps](https://geth.ethereum.org/docs/developers/dapp-developer/native-bindings) page for details.                                  |
|   `evm`    | Developer utility version of the EVM (Ethereum Virtual Machine) that is capable of running bytecode snippets within a configurable environment and execution mode. Its purpose is to allow isolated, fine-grained debugging of EVM opcodes (e.g. `evm --code 60ff60ff --debug run`).                                                                                                                                                                                                                                               |
| `rlpdump`  | Developer utility tool to convert binary RLP ([Recursive Length Prefix](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp)) dumps (data encoding used by the Ethereum protocol both network as well as consensus wise) to user-friendlier hierarchical representation (e.g. `rlpdump --hex CE0183FFFFFFC4C304050583616263`).                                                                                                                                                                                |

## Running `geth`

Going through all the possible command line flags is out of scope here (please consult our
[CLI Wiki page](https://geth.ethereum.org/docs/fundamentals/command-line-options)),
but we've enumerated a few common parameter combos to get you up to speed quickly
on how you can run your own `geth` instance.

### Hardware Requirements

Minimum:

* CPU with 4+ cores
* 8GB RAM
* 1TB free storage space to sync the Mainnet
* 8 MBit/sec download Internet service

Recommended:

* Fast CPU with 8+ cores
* 16GB+ RAM
* High-performance SSD with at least 1TB of free space
* 25+ MBit/sec download Internet service

### Full node on the main Ethereum network

By far the most common scenario is people wanting to simply interact with the Ethereum
network: create accounts; transfer funds; deploy and interact with contracts. For this
particular use case, the user doesn't care about years-old historical data, so we can
sync quickly to the current state of the network. To do so:

```shell
$ geth console
```

This command will:
 * Start `geth` in snap sync mode (default, can be changed with the `--syncmode` flag),
   causing it to download more data in exchange for avoiding processing the entire history
   of the Ethereum network, which is very CPU intensive.
 * Start the built-in interactive [JavaScript console](https://geth.ethereum.org/docs/interacting-with-geth/javascript-console),
   (via the trailing `console` subcommand) through which you can interact using [`web3` methods](https://github.com/ChainSafe/web3.js/blob/0.20.7/DOCUMENTATION.md) 
   (note: the `web3` version bundled within `geth` is very old, and not up to date with official docs),
   as well as `geth`'s own [management APIs](https://geth.ethereum.org/docs/interacting-with-geth/rpc).
   This tool is optional and if you leave it out you can always attach it to an already running
   `geth` instance with `geth attach`.

### A Full node on the Holesky test network

Transitioning towards developers, if you'd like to play around with creating Ethereum
contracts, you almost certainly would like to do that without any real money involved until
you get the hang of the entire system. In other words, instead of attaching to the main
network, you want to join the **test** network with your node, which is fully equivalent to
the main network, but with play-Ether only.

```shell
$ geth --holesky console
```

The `console` subcommand has the same meaning as above and is equally
useful on the testnet too.

Specifying the `--holesky` flag, however, will reconfigure your `geth` instance a bit:

 * Instead of connecting to the main Ethereum network, the client will connect to the Holesky 
   test network, which uses different P2P bootnodes, different network IDs and genesis
   states.
 * Instead of using the default data directory (`~/.ethereum` on Linux for example), `geth`
   will nest itself one level deeper into a `holesky` subfolder (`~/.ethereum/holesky` on
   Linux). Note, on OSX and Linux this also means that attaching to a running testnet node
   requires the use of a custom endpoint since `geth attach` will try to attach to a
   production node endpoint by default, e.g.,
   `geth attach <datadir>/holesky/geth.ipc`. Windows users are not affected by
   this.

*Note: Although some internal protective measures prevent transactions from
crossing over between the main network and test network, you should always
use separate accounts for play and real money. Unless you manually move
accounts, `geth` will by default correctly separate the two networks and will not make any
accounts available between them.*

### Configuration

As an alternative to passing the numerous flags to the `geth` binary, you can also pass a
configuration file via:

```shell
$ geth --config /path/to/your_config.toml
```

To get an idea of how the file should look like you can use the `dumpconfig` subcommand to
export your existing configuration:

```shell
$ geth --your-favourite-flags dumpconfig
```

#### Docker quick start

One of the quickest ways to get Ethereum up and running on your machine is by using
Docker:

```shell
docker run -d --name ethereum-node -v /Users/alice/ethereum:/root \
           -p 8545:8545 -p 30303:30303 \
           ethereum/client-go
```

This will start `geth` in snap-sync mode with a DB memory allowance of 1GB, as the
above command does.  It will also create a persistent volume in your home directory for
saving your blockchain as well as map the default ports. There is also an `alpine` tag
available for a slim version of the image.

Do not forget `--http.addr 0.0.0.0`, if you want to access RPC from other containers
and/or hosts. By default, `geth` binds to the local interface and RPC endpoints are not
accessible from the outside.

### Programmatically interfacing `geth` nodes

As a developer, sooner rather than later you'll want to start interacting with `geth` and the
Ethereum network via your own programs and not manually through the console. To aid
this, `geth` has built-in support for a JSON-RPC based APIs ([standard APIs](https://ethereum.github.io/execution-apis/api-documentation/)
and [`geth` specific APIs](https://geth.ethereum.org/docs/interacting-with-geth/rpc)).
These can be exposed via HTTP, WebSockets and IPC (UNIX sockets on UNIX based
platforms, and named pipes on Windows).

The IPC interface is enabled by default and exposes all the APIs supported by `geth`,
whereas the HTTP and WS interfaces need to manually be enabled and only expose a
subset of APIs due to security reasons. These can be turned on/off and configured as
you'd expect.

HTTP based JSON-RPC API options:

  * `--http` Enable the HTTP-RPC server
  * `--http.addr` HTTP-RPC server listening interface (default: `localhost`)
  * `--http.port` HTTP-RPC server listening port (default: `8545`)
  * `--http.api` API's offered over the HTTP-RPC interface (default: `eth,net,web3`)
  * `--http.corsdomain` Comma separated list of domains from which to accept cross-origin requests (browser enforced)
  * `--ws` Enable the WS-RPC server
  * `--ws.addr` WS-RPC server listening interface (default: `localhost`)
  * `--ws.port` WS-RPC server listening port (default: `8546`)
  * `--ws.api` API's offered over the WS-RPC interface (default: `eth,net,web3`)
  * `--ws.origins` Origins from which to accept WebSocket requests
  * `--ipcdisable` Disable the IPC-RPC server
  * `--ipcpath` Filename for IPC socket/pipe within the datadir (explicit paths escape it)

You'll need to use your own programming environments' capabilities (libraries, tools, etc) to
connect via HTTP, WS or IPC to a `geth` node configured with the above flags and you'll
need to speak [JSON-RPC](https://www.jsonrpc.org/specification) on all transports. You
can reuse the same connection for multiple requests!

**Note: Please understand the security implications of opening up an HTTP/WS based
transport before doing so! Hackers on the internet are actively trying to subvert
Ethereum nodes with exposed APIs! Further, all browser tabs can access locally
running web servers, so malicious web pages could try to subvert locally available
APIs!**

### Operating a private network

Maintaining your own private network is more involved as a lot of configurations taken for
granted in the official networks need to be manually set up.

Unfortunately since [the Merge](https://ethereum.org/en/roadmap/merge/) it is no longer possible
to easily set up a network of geth nodes without also setting up a corresponding beacon chain.

There are three different solutions depending on your use case:

  * If you are looking for a simple way to test smart contracts from go in your CI, you can use the [Simulated Backend](https://geth.ethereum.org/docs/developers/dapp-developer/native-bindings#blockchain-simulator).
  * If you want a convenient single node environment for testing, you can use our [Dev Mode](https://geth.ethereum.org/docs/developers/dapp-developer/dev-mode).
  * If you are looking for a multiple node test network, you can set one up quite easily with [Kurtosis](https://geth.ethereum.org/docs/fundamentals/kurtosis).

## Contribution

Thank you for considering helping out with the source code! We welcome contributions
from anyone on the internet, and are grateful for even the smallest of fixes!

If you'd like to contribute to go-ethereum, please fork, fix, commit and send a pull request
for the maintainers to review and merge into the main code base. If you wish to submit
more complex changes though, please check up with the core devs first on [our Discord Server](https://discord.gg/invite/nthXNEv)
to ensure those changes are in line with the general philosophy of the project and/or get
some early feedback which can make both your efforts much lighter as well as our review
and merge procedures quick and simple.

Please make sure your contributions adhere to our coding guidelines:

 * Code must adhere to the official Go [formatting](https://golang.org/doc/effective_go.html#formatting)
   guidelines (i.e. uses [gofmt](https://golang.org/cmd/gofmt/)).
 * Code must be documented adhering to the official Go [commentary](https://golang.org/doc/effective_go.html#commentary)
   guidelines.
 * Pull requests need to be based on and opened against the `master` branch.
 * Commit messages should be prefixed with the package(s) they modify.
   * E.g. "eth, rpc: make trace configs optional"

Please see the [Developers' Guide](https://geth.ethereum.org/docs/developers/geth-developer/dev-guide)
for more details on configuring your environment, managing project dependencies, and
testing procedures.

### Contributing to geth.ethereum.org

For contributions to the [go-ethereum website](https://geth.ethereum.org), please checkout and raise pull requests against the `website` branch.
For more detailed instructions please see the `website` branch [README](https://github.com/ethereum/go-ethereum/tree/website#readme) or the 
[contributing](https://geth.ethereum.org/docs/developers/geth-developer/contributing) page of the website.

## License

The go-ethereum library (i.e. all code outside of the `cmd` directory) is licensed under the
[GNU Lesser General Public License v3.0](https://www.gnu.org/licenses/lgpl-3.0.en.html),
also included in our repository in the `COPYING.LESSER` file.

The go-ethereum binaries (i.e. all code inside of the `cmd` directory) are licensed under the
[GNU General Public License v3.0](https://www.gnu.org/licenses/gpl-3.0.en.html), also
included in our repository in the `COPYING` file.

