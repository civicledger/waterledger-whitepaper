![WaterLedger Screenshot](https://waterledger-wp.sgp1.digitaloceanspaces.com/waterledger-screenshot.jpg)

# Operation of WaterLedger

## Registration

Access to the WaterLedger platform is only possible by prior registration, which as part of the pilot is done manually. Each participant in the pilot scheme has been added to a database, which also includes their Water Account details and available allocations.

The WaterLedger application then shows the user that they are not logged in. It is able to determine this by checking the user’s local storage for relevant keys.

An unauthenticated user is then able to search for any one of their Water Account numbers in a form at the top of the homepage. For privacy reasons they must also enter a code that is provided directly by the scheme organiser. The search will then retrieve their user details and water account details.

The user may then “claim” the accounts. Doing so then creates an Ethereum Wallet, with accounts corresponding to each Water Trading Zone. These Ethereum account details and the associated water trading licence information are stored in the Users smart contract, which handles all aspects of permissioning and identification.

Critical details (especially IDs) are stored in the browser’s local storage to facilitate later retrieval.

## Selecting a Water Account

The user’s water licences/accounts are visible in the top right. The first account is set as the default for usage. However, they can at any time click on a different account and set that one as the used account.

## Placing a Buy or Sell Order

Once registered, a user has access to place either a buy (bid) or sell (offer) order to or from the active account. Clicking on the buttons in the buy and sell listing interface enable a pop-up modal window with a form containing the price and quantity. In a bid order there is also a temporary allocation transfer period.

When a sell order is placed, the balance to be sold is immediately withdrawn from the allocation of the zone the user is selling in. This prevents over-selling of the water allocation.

There is no equivalent balance reduction for buyers, who may put whatever trades they think are beneficial.

## Matching

The OrderBook smart contract will sort all orders by their relevant weighting. If a buy order is placed it will sort the sell orders to the lowest first. If a sell order is placed it will sort the buy orders to the highest first. It then checks against their total volumes. It will buy or sell against those volumes until it matches the required amounts to fill the order.

Though the OrderBook matching will buy out multiple orders to place an order it will **not**  partially buy or split order volume.

If an order cannot be successfully and fully matched it is placed on the OrderBook without matching.

Successful creation of a match, and thereby a trade ostensibly requires approval from the water trading authority. In practical terms, however, that step is skipped, and all trades are considered pre-approved. This will be subject to change depending on the requirements of the water trading authority.

If an order matches successfully it then updates the LastTradedPrice statistic accordingly.

## History and Trade Creation

As a match is created, it adds an entry to the History smart contract, which stores a list of “trades”. These trades are able to be retrieved directly from the smart contract, and will automatically appear in the order book under the buy and sell orders, or on a separate page dedicated to the fuller trade history.

## Liability Creation

The process of adding a trade to the history triggers a `HistoryAdded` event that contains all of the relevant data for the trade. This is watched by the WaterLedger NodeJS API which has a websocket connection to the History contract. When that event is triggered it begins the DAML portion of the workflow.

*This section of the workflow is somewhat tentative and is still being finalised.*

The API deploys a contract to the Ledger which is called a TradeProposal. This is due to DAML’s inherent inability to create a liability on someone else’s behalf without them approving it. Once a TradeProposal is created, the buyer is notified of the transaction, and can approve it. This then triggers the creation of a NewTrade from the details already provided. Once this is done the remainder of the workflow, specifically the callback from a payment gateway of some kind, is synthesised.

## Trade Completion

Once the transaction is “completed” by the liability system, the OrderBook smart contract is updated. It finalises the transfer of the Water tokens into the zone that they were purchased into.

Note that this may mean tokens are `burned` in one zone, and then `minted` into another.

# Architecture Components

![WaterLedger Screenshot](https://waterledger-wp.sgp1.digitaloceanspaces.com/Architecture.png)

## Frontend Single Page App

The JavaScript application that the user interacts with directly. This is the "face" of the system.

The Single Page App is directly connected to the Ethereum blockchain, and implements its own wallet and websocket event management.

## Ethereum Smart Contracts

Contracts written in Solidity are the primary persistence store for order book and trade history. The smart contracts are documented more extensively in [their own repository](https://github.com/civicledger/waterledger-contracts).

Users may push orders to the orderbook smart contract, and the matches that occur will trigger events, most critically the `HistoryAdded` event, which are watched for by other systems.

## DAML Contracts

DAML contracts form the liability between the buyer and the seller. The DAML workflow also triggers websocket events.

## API

The API has minimal functionality by design. It stores minimal data in a MongoDB database, but the primary datasource is the Ethereum Smart contracts.

It has two main roles. One is to store and serve the required ABI and address locations for multiple potential instances of the waterledger contracts. The other is to act as a websocket watcher for events from both the DAML ledger and the Ethereum network. This allows the API to trigger workflow steps.

# Limits of Solidity and Ethereum

One of WaterLedger’s core goals was to truly use the blockchain. This meant not providing a thin veneer of blockchain transaction on a traditional web application, but using the EVM as a the sole persistence layer.

Though this vision has come with critical benefits, it has tended to expose limitations and complexities within the core language, libraries and ecosystem.

## Solidity

Since starting core development in 2018 Solidity has been through three point version changes, from 0.4.13 though to the current 0.6.2. Most of these changes have made Solidity more strict and explicit, but this has come with challenges from a development perspective.

### Array Handling

One of the most challenging elements of Solidity is its limited support for arrays. Though elements can be stored in arrays, the purely imperative nature of array handling means concepts like sorting and filtering - trivial in other languages - can become extremely difficult.

In particular there is a need to run imperative loops over data, often multiple times. Due to the presence of “gas”, this can lead to a high execution cost.

For example, attempting to retrieve all of the trades for a specific zone necessitates running a loop on all of the trades, and incrementing a counter to determine the number of matching trades. This then can be used to find the correct number trades, to create the array (which needs to be created with a set length). Only then can the code loop yet again through the trades, populating the return array.

There are similar issues with sorting data, for example by price.  Sorting an array by a given field in most languages is a trivial process, however Solidity requires 15 lines of expensive code per sortable field.

Though a call that only retrieves ledger data has no gas cost, there is a significant maintenance and development cost associated with this highly explicit, imperative, and error-prone code.

### Shortage of Information

Much of the knowledge around Solidity is based on transfer of basic tokens, or trivial book-keeping and reconciliation. Knowledge about more complex interactions like structs inside arrays, or vice versa, or better techniques for unit testing or deployment… this sort of knowledge is hard-won. Use of Ethereum as an application platform, as opposed to a basic token, is largely an untrodden path, and it is loaded with challenges and complexities.

This makes it tempting to defer or offload more challenging features, such as using an API to store the “real” data, but reconciling it against the blockchain, or returning all data, but filtering in the client.

Like many technologies, knowledge beyond Solidity 101 tends to be sparse, and where it does exist it often means implementing more advanced libraries or approaches (such as DeFi, oracles, etc) rather than more advanced use of the language itself.

## Tooling

### General Immaturity of tooling

Though it has gotten somewhat better over the course of the last year, the blockchain market is riddled with beta (or alpha) tools and libraries. Fundamental, core technologies such as Web3, Metamask and Truffle, have substantially changed their APIs and operation, each time imposing a varied burden in development and learning.

### Managing of ABI and Address

The previously mentioned Truffle is a core technology for Solidity and the Ethereum ecosystem. It provides the leading tools for development and testing of smart contracts, as well as Ganache, a blockchain test node.

One of the primary roles Truffle takes is that of deployment. Both in local development and in live deployment Truffle’s deployment tools can be used to create and store live instances of the smart contract.

In particular, it creates a file typically called a “truffle file” this file is a large JSON file that contains the outputs from compiling the smart contract, as well as  the address of the deployed contract.

This creates a number of issues. First of all, the file is extremely large, at approximately 3 megabytes. Significantly larger alone than the rest of a single page application needs to be. This is because it contains large amounts of extraneous information in a highly inefficient format.  The majority of this is an AST, the output of compiling. This has no value in a web context, and is actually duplicated, accounting for tens of thousands of lines and several megabytes.

To overcome this we have had to create our own software, a library that opens the artefact file, then removes the unnecessary elements, and compresses the result, for around a 90% size reduction. This library has been open-sourced to the community.

The other issue, however, is that the truffle stores the contract’s location keyed against its network ID. This means that if multiple instances of the contract need to be deployed to the same network, Truffle will replace it. As WaterLedger needs to store multiple instances (the Zone contract alone is deployed five times) Truffle’s deployment process was not suitable.

In order to resolve this we have had to create a custom solution, creating an API that facilitates deployment, storing the contract address in a database, and allowing retrieval of the ABI and address by a standard HTTP request.

## The EVM

### Gas Costs

The presence of gas is an interesting idea for Ethereum. It means that while  network access is free in terms of infrastructure cost, it is an unknown cost in terms of ongoing operational costs.  Note that “gas” has a cost translatable to Ether, which itself has a direct cost in fiat currency.

Gas costs are something of a “dark art”, notoriously difficult to calculate with accuracy. Sloppy, thoughtless, or even seemingly perfectly valid code can increate gas execution costs, and any addition of functionality naturally incurs a corresponding increase in gas costs. Optimising against these costs is an ongoing challenge, requiring specialist knowledge.

Gas costs also apply to the deployment process, and at a scale many orders of magnitude higher than execution costs. Though deployment can be more easily managed, the cost scales directly with functionality.

Gas costs also scale with the cost of gas. Ether is a volatile cryptocurrency, that has enjoyed or suffered hundreds of percent increases and drops over the last few years. This shifting target makes calculations challenging.

### The need for a positive Ether balance

Though this might seem like the above, the very fact that users need any amount of gas (ether) in their account at all is an incredible impediment to any on-boarding process.

WaterLedger delays this issue by providing a small amount of ether to user accounts as part of their account creation.  However, this solution is a crutch; if too much ether is sent it is wasted, but insufficient will result in users being unable to transact. This then means developing further functionality to manage and watch and alert for user’s ether balance.

A solution exists in the form of the Gas Station Network, which lets system operators take responsibility for gas costs on behalf of the user. However, the addition of complexity for support of this network, as well as the fees for the network itself, will add to both transaction and deployment costs.

Note that gas costs are dealt with in great detail in a later section.

# Technology Choices

The layers that form WaterLedger’s technology choices were carefully selected. Like most technology choices, many platforms or solutions would have potentially worked, with various pros and cons contributing to a final selection.

WaterLedger is built on a stack of four separate layers.

![WaterLedger Screenshot](https://waterledger-wp.sgp1.digitaloceanspaces.com/logos.png)

## FrontEnd Single Page Application

### Requirement

The SPA component of WaterLedger is the visible frontend for users. It provides the interface that displays orders and statistics and allows users to interact and place orders. There are no options in this space beyond JavaScript (or potentially languages that compile to JavaScript) but there are many capable options within this space.

One of the key requirements is that the frontend SPA does not connect directly to the API for persistence as is the most typical approach. It actually connects directly to the blockchain. This means the solution has to be able to implement wallet handling, authentication, and all storage, retrieval and state management.

### Selection: React

Though there are a large number of options, React was the number one choice. This was mostly because React enjoys the most support within blockchain development tooling and established community knowledge. Trying to port that knowledge and tooling into Angular or Ember (for example) would not have been a viable use of resources.

## API and Database

### Requirement

WaterLedger requires an API but does not use the API for significant persistence of orderbook or trade data. The primary purpose of the API is to store contract deployment details, and provide a mechanism for retrieval of address and ABI. This allows the React application to form a valid contract instance to query and transact on.

The application also handles some degree of “workflow” processing, tracking contract events to trigger further changes.

### Selection: NodeJS and MongoDB

The requirement for a small, light, and fast system with good support for WebSocket connections made NodeJS an obvious choice. For the persistence layer, the document-centric nature of contract deployments made MongoDB an excellent fit.

This combination creates a fast, agile application that can be deployed anywhere from serverless environments like Lambda to scalable cloud instances.

## Blockchain or DLT

### Requirements

Given WaterLedger is primarily a DLT project, the choice of DLT provider and platform is a critical one. WaterLedger as a platform has three primary requirements. First, it needs to store orders. Orders are a request to buy a certain amount of Water (stored as an ERC-20 token) for a given amount. These are placed on the order book and can be matched against. Secondly, when matching orders are placed, it needs to pair a buy and sell order, and create and store the trade. Finally, it needs to create store the financial liability created by the trade, essentially the “buy” side of the transaction.

A large part of the purpose of WaterLedger is to create a near-realtime trading platform. Near-realtime is a subjective term, but the current process is one of days or weeks, and we aim for minutes. Another key requirement is that the system be explicitly open and transparent. The water market has been plagued with closed access to data, and some potentially suspect trade. An open and free market benefits all honest actors.

While the process of placing orders and even more so finalised trades is critically open and transparent for accountability, the financial relationship between buyer and seller is not a matter of public interest.

### Selection: Ethereum and elements of DAML

The primary platform used is the public Ethereum network. There are a number of reasons for this, but many of them are overshadowed by the simple lack of options at the outset of this project. Regardless, even at the current state of blockchain and DLT development there are no platform that better match our requirements. Many existing options are too experimental and early to support our needs. Many are alphas, lack mature tooling or knowledge, and require high level knowledge of a general purpose language.

One of the few platforms we examined with any significant success was DAML by Digital Asset. Though a niche technology even in DLT terms, it provides an impressive set of features, many of which outperform Ethereum.

As a platform option, DAML provided a compelling case, especially in terms of searching and filtering data, but a lack of transparency and the requirement for a trustless paradigm meant we stayed with Ethereum for public-facing code. As a proof of concept, however, liability management and payment processing - all of which is confidential is handled in DAML.

This question of trust, accountability and transparency is a key issue for WaterLedger as a platform and deserves further discussion.

# Trustlessness and Transparency
The primary requirements of WaterLedger stem from a need for a trusted and open source of information.  While it would in theory be possible for WaterLedger to become a trusted authority,  the truest trust comes from trustlessness.

Rather than requiring users, regulators, and the public to trust Waterledger to be upfront in its data provision, we prefer that instead we use a storage mechanism that means information cannot possibly be omitted. This simply eliminates trust as a factor.

The public Ethereum network cannot be modified, and data cannot be omitted or occluded in any way. This is why Waterledger uses the public Ethereum network as its primary persistence layer. More particularly this is why WaterLedger does not rely on a traditional database, nor on any of the alternative blockchain options.

In particular we investigated Hyperledger Fabric, DAML, and Enterprise Ethereum. These offered significant advantages over the public Ethereum network. In particular, the elimination of “gas” costs eased the on-boarding process greatly. This is a notable disadvantage of the public Ethereum network, as it means a small amount of cryptocurrency has to be “seeded” into each user’s account, and monitored for overruns, etc.

DAML also offers impressive features for searching for and filtering existing contracts and data using a REST API which would greatly alleviate the complexities involved in doing this in Solidity (see previous documentation).

However, these options all provide a similar flaw - they are not trustless. Enterprise Ethereum and Hyperledger can have data visibility configured in detail, while DAML is completely opaque by default and access needs to be explicitly enabled.
