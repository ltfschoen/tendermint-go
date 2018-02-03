# Installation of GoLang and Tendermint

* Install Go and Tendermint
* Check versions installed
  ```bash
  $ go version
  go version go1.9.2 darwin/amd64
  $ tendermint version
  0.15.0
  $ abci-cli version
  0.9.0
  ```

# Running a Modified version of the "Dummy" ABCI Application used with Tendermint ABCI-CLI

* Clone **[https://github.com/tendermint/abci](https://github.com/tendermint/abci)**.
  ```bash
  go get github.com/tendermint/abci && cd $GOPATH/src/github.com/tendermint/abci
  ``` 

* Install Dependencies and Update Build of `abci-cli` Executable in /Users/Username/go/bin/abci-cli
  ```bash
  make get_vendor_deps && make install
  ```

* Modify the Source Code of Tendermint Go "Dummy" ABCI Application

  * Edit the `Info` Method. Add Debugging with `fmt.Printf`
  ```go
  func (app *DummyApplication) Info(req types.RequestInfo) (resInfo types.ResponseInfo) {
    fmt.Printf("DummyApplication Info method with req %v", req)
    return types.ResponseInfo{Data: fmt.Sprintf("{\"size\":%v}", app.state.Size())}
  }
  ```

* Re-Build the Tendermint Go "Dummy" ABCI Application

  * **/Users/Username/go/src/github.com/tendermint/abci/example/dummy/dummy.go**

    * About: Uses an [Immutable AVL+ (IAVL) Merkle Tree](https://github.com/tendermint/iavl) and an [implemention of LevelDB in Go](https://github.com/tendermint/tmlibs/) to [store transactions in a Merkle Tree](https://github.com/tendermint/abci/blob/master/example/dummy/README.md)
    * Re-Build with Make
    ```bash
    cd $GOPATH/src/github.com/tendermint/abci && make get_vendor_deps && make install
    ```

* Inspecting the ABCI-CLI "Socket" Client

  * **/Users/Username/go/src/github.com/tendermint/abci/client/client.go**

    * ABCI-CLI Client Interface Definitions 
      * `__Async` methods return `ReqRes` object
      * `__Sync` methods return ProtoBuf `Response__` struct and errors
        * Note: Errors are due to ABCI Client socket connectivity issues
        * Note: Application-specific errors are included in ABCI Error Codes and Logs of a response
    
    * `NewClient` method returns New ABCI-CLI Client of given Transport Type (i.e. `"socket"` or `"grpc"`)
      * `NewSocketClient` method in **/client/socket_client.go** is called for given Transport Type of `"socket"`, which creates a new Base Service for the ABCI-CLI "Socket" Client.

* Run ABCI-CLI "Socket" Client to Connect to "Socket" Server associated with "Dummy" ABCI Application and Send ABCI Messages to return a Response

  * **/Users/Username/go/src/github.com/tendermint/abci/cmd/abci-cli/abci-cli.go**

    * Interaction with the ABCI "Socket" Client using the ABCI-CLI interpreter /Users/Username/go/src/github.com/tendermint/abci/cmd/abci-cli/abci-cli.go

      * Start ABCI-CLI "Socket" Client, which opens a new connection to the Go "Socket" Server associated with the Go "Dummy" ABCI Application
        ```bash
        abci-cli dummy --verbose --abci="socket" --address="tcp://0.0.0.0:46658"

        Starting ABCIServer            module=abci-server impl=ABCIServer
        Waiting for new connection...  module=abci-server 
        Accepted a new connection      module=abci-server
        ```

      * View 

      * Initialise Tendermint Configuration 
        ```bash
        tendermint init --trace
        ```

      * View Tendermint Node Configuration
        ```bash
        cat /Users/Username/.tendermint/config.toml
        cat /Users/Username/.tendermint/genesis.json
        ```

      * Start the Tendermint Node with a One-Node Blockchain with Reset 
        ```bash
        tendermint unsafe_reset_all
        tendermint node --trace

        Executed block                 module=state height=17 validTxs=0 invalidTxs=0
        Committed state                module=state height=17 txs=0 appHash=
        Executed block                 module=state height=18 validTxs=0 invalidTxs=0
        Committed state                module=state height=18 txs=0 appHash=
        ```

      * Send an ABCI Message to a Handler (that conforms to the ABCI Interface Specification) that calls a Method specific to the Go "Dummy" ABCI Application and Wait for Response (i.e. `abci-cli info` calls the `Info()` method of the app and responds with the quantity of transactions in the Merkle Tree)
        ```bash
        abci-cli info
        ```

      * View the Custom Debugging in the Server Terminal Logs

      * Send Transaction with cURL to the "Dummy" ABCI Application of the Tendermint Node. Transaction contains bytes "abcd" to be stored as both key and value in the Merkle tree.
        ```bash
        curl -s 'localhost:46657/broadcast_tx_commit?tx="abcd"'

        {
          "jsonrpc": "2.0",
          "id": "",
          "result": {
            "check_tx": {
              "code": 0,
              "data": "",
              "log": "",
              "gas": "0",
              "fee": "0"
            },
            "deliver_tx": {
              "code": 0,
              "data": "",
              "log": "",
              "tags": [
                {
                  "key": "app.creator",
                  "valueType": 0,
                  "valueString": "jae",
                  "valueInt": "0"
                },
                {
                  "key": "app.key",
                  "valueType": 0,
                  "valueString": "abcd",
                  "valueInt": "0"
                }
              ]
            },
            "hash": "2B8EC32BA2579B3B8606E42C06DE2F7AFA2556EF",
            "height": 1152
          }
        }
        ```

        * Triggers the `OnStart` function in the Go ABCI "Socket" Server file **/server/socket_server.go**, which calls its `acceptConnectionsRoutine` function, which calls its `handleRequests` function that reads requests from its signal connection and passes the channel that buffers responses to it. It performs a Mutex Lock before calls its `handleRequest` 

      * Send Query with cURL to the "Dummy" ABCI Application of the Tendermint Node. Verifies if the Transaction was successful and that bytes "abcd" were stored as both key and value in the Merkle tree.
        ```bash
        curl -s 'localhost:46657/abci_query?data="abcd"'
        {
          "jsonrpc": "2.0",
          "id": "",
          "result": {
            "response": {
              "code": 0,
              "index": "-1",
              "key": "61626364",
              "value": "61626364",
              "proof": "010114801633CA20909B4374AE0471AC9152588741CFAB00000000000000010100",
              "height": "0",
              "log": "exists"
            }
          }
        }
        ```

      * Send Multiple ABCI Messages over a Single Connection using the `abci-cli console` and `abci-cli batch` commands

# GoLang

## Learning Progress

* [X] - Learn Go in Y Minutes (Quick Refresher) - https://learnxinyminutes.com/docs/go/
* [X] - Go "Pass by Pointer (Reference)" vs "Pass by Value" - http://goinbigdata.com/golang-pass-by-pointer-vs-pass-by-value/
* [X] - GoLang Book - Concurrency - Blocking, Goroutines and Synchronous Unbuffered Channels (allows twos Goroutines to communicate and synchronise execution), Channel Directions (i.e. Send-Only, Receive-Only, Bi-directional), Select (for Channels), Asynchronous Buffered Channels (with Capacity) - https://www.golang-book.com/books/intro/10
* [X] - GoLang Book - Core Packages - Mutex (protection shared resources from the interuption of Non-Atomic operations), Servers (RPC) - https://www.golang-book.com/books/intro/13#section6
* [ ] - GoLang Tutorials - Object-Oriented Programming (O-O) (Associated methods with a Struct), Encapsulation and Package Visibility (capitalising first letter) - http://golangtutorials.blogspot.com.au/2011/05/table-of-contents.html
* [ ] - GoLang Book (An Introduction to Programming in Go) - https://www.golang-book.com/books/intro
* [ ] - Go By Example - https://gobyexample.com

# References:

* Tendermint Getting Started Guide - http://tendermint.readthedocs.io/projects/tools/en/master/getting-started.html
* Tendermint Architecture Decision Records - `cd /Users/Ls/go/src/github.com/tendermint/tendermint/docs/architecture && ls -al`

* GoLang Learning Guides - https://github.com/golang/go/wiki/Learn
* Go REPL - https://play.golang.org/
* Go Official Docs - https://golang.org/doc/
* Go Functions of Standard Library (with Examples) - https://golang.org/pkg/