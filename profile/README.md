# StellarBroker

> [StellarBroker](https://stellar.broker) is a multi-source liquidity swap router for Stellar, providing access to
AMMs like SoroSwap and Aquarius, Stellar DEX, and Stellar Classic AMMs.

## How It Works 
![stellar-broker-simple-diagram](https://github.com/user-attachments/assets/6640f0be-6767-405e-9110-4798b92e60e9)

## Technical Architecture

**StellarBroker Router Server** consists of client gateways (REST API + WebSocket), matcher engine that operates with
in-memory blockchain state graphs, transaction execution pipeline, ledger state reader feeding updates from Stellar Core
to the transaction result processor, and MongoDB-backed swaps history storage.

![stellar-broker-architecture-diagram](https://github.com/user-attachments/assets/4d824f66-5cc5-44a9-b680-81fe10e3b087)

**Ledger State Reader** fetches blockchain state directly from a Stellar Core node and feeds the data to the in-memory
**DEX Graph** and **Soroban Graph** through the **Graph Loader** which relies on the set of protocol-specific providers
that parse orderbook and AMM state.

**Matcher** replicates the entire path finding and trade execution engine from Stellar Core and Horizon.
The key distinction of our implementation lies in the ability to split the quote amount in smaller trades and find all
possible trade routes in parallel. For example, Horizon offers conversion paths to trade either through
DEX or Classic AMMs, while our matcher taps liquidity of both these sources, extending it even further with
the liquidity of Soroban AMMs. Having the blockchain orderbooks + AMMs graph state in memory provides 
an unmatched performance, allowing to perform tens of thousands of quote simulations per second to find the combination
of the best routes to execute the quote.

**Shard Dispatcher** further boosts server performance by maintaining a copy of state graph per each CPU core in
a separate thread in order to increase the number of parallel connections.

Applications (wallets and trading interfaces) interact with the server using a client library that connects to
the server via **WebSocket Gateway** or REST **API Gateway**. **Trade Broker** is responsible for maintaining swap sessions,
price quotation, and trade execution. Following the quote request params passed from a client app, it obtains a quote
from **Matcher**, prepares a set of transactions for trading through different routes and sends it back to the client,
which in turn verifies and sends signed transactions to the server. Although somewhat complicated, this authorization
flow provides a non-custodial service with a CEX-like user experience, while ensuring the security of user funds and
guaranteed quote execution.

**Transaction Submitter** simulates and submits transactions to Stellar Network in parallel. For large volume swaps
it’s important to execute all quote trades in parallel, in the same ledger. Otherwise arbitrage bots may intervene,
cutting potential savings for the end-users. To achieve this, all quoted transactions originate from different channel
accounts managed by **Channel Pool** and get submitted to the network simultaneously.

Once the transaction is executed, **TxResult Processor** retrieves the actual bought/sold amount and stores this
information in the database. Subsequently, **Trade Broker** module notifies the client about current swap progress status
and prepares additional transactions if the quote has not been executed in full in the latest ledger.
