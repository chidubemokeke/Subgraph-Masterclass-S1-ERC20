# Subgraph Development Masterclass - 1

Welcome to the Beginner Level of the Subgraph Development Masterclass. This tutorial will guide you through the fundamentals of subgraph development using The Graph Studio, from setting up your environment to deploying your first subgraph.

## Overview

In this masterclass, you'll gain essential skills in subgraph development, enabling you to efficiently index and query blockchain data for your decentralized application(dApps).

## Learning Objectives

By completing this masterclass, you will:

- Understand the purpose and benefits of subgraphs in dApps.
- Learn how to set up your development environment for subgraph development.
- Define GraphQL entities to model data structures for indexing smart contract events.
- Implement mappings logic to transform blockchain events to entities in your subgraph.
- Deploy your subgraph using The Graph Studio for querying.

## Prerequisites

Before you begin, ensure you have the following installed:

- [Node.js](https://nodejs.org/)
- [Yarn](https://classic.yarnpkg.com/en/docs/install)
- [Graph CLI](https://github.com/graphprotocol/graph-cli)
- [The Graph Studio Account](https://thegraph.com/studio/)

## Step-by-Step Tutorial

### Step 1: Using The Graph Studio

**Create a New Subgraph in The Graph Studio:**

- Navigate to The Graph Studio, sign in, and create a new subgraph.
- Enter the required details such as the subgraph name and description.
- you will receive a deploy key and subgraph slug (e.g., username/subgraph-name)
- Initialize your subgraph with the command from the studio.

```bash
graph init --studio <subgraph-name>
```

**Authenticate with The Graph Studio:**

- Copy the deploy key from The Graph Studio
- Run the following command in your terminal to authenticate:

```bash
graph auth --studio <ACCESS_TOKEN>
```

### Step 2: Understanding the Project Structure

The initialized project contains the following structure:

- **subgraph.yaml**: Defines the subgraph manifest.
- **schema.graphql**: Contains the GraphQL schema.
- **src/**: Contains the source files.
  - **mappings/**: Contains the mapping functions.
    - **uniswap.ts**: Handles the mappings for the Uniswap contract.
  - **utils/**: Contains utility files.
    - **helper.ts**: Contains helper functions.
- **abis/**: Contains the contract ABI files.
  - **Uniswap.json**: ABI for the Uniwsap contract.
- **generated/**: Contains auto-generated files.
  - **schema.ts**: Type definitions generated from the GraphQL schema.

### Step 3: Defining GraphQL Entities

In subgraph development, GraphQL entities define the data structures that your subgraph will index from Ethereum smart contracts. Let's break down the entities defined in schema.graphql:

```gql
type Transfer @entity(immutable: true) {
  id: ID!
  from: Account!
  to: Account!
  value: BigInt!
  timestamp: BigInt!
}

type Account @entity {
  id: ID!
  sentTransfers: [Transfer!]! @derivedFrom(field: "from")
  receivedTransfers: [Transfer!]! @derivedFrom(field: "to")
  totalSent: BigInt!
  totalReceived: BigInt!
  sentCount: Int!
  receivedCount: Int!
}
```

#### Transfer Entity

- Purpose: The Transfer entity represents a transfer event on the Ethereum blockchain.
- Fields:
  - id: A unique identifier for each transfer event, combining the transaction hash and log index.
    - from: A reference to the Account entity representing the sender.
    - to: A reference to the Account entity representing the receiver.
    - value: The amount transferred, represented as a BigInt.
    - timestamp: The block timestamp when the transfer occurred, represented as a BigInt.
    - @entity(immutable: true): This directive ensures that the entity's fields cannot be modified once created, reflecting the immutable nature of blockchain data.

#### Account Entity

- Purpose: The Account entity represents an Ethereum account involved in transfers.
- Fields:
  - id: A unique identifier for each account, usually the account address.
  - sentTransfers: A list of Transfer entities where this account is the sender.
  - receivedTransfers: A list of Transfer entities where this account is the receiver.
  - totalSent: The total amount sent by this account, represented as a BigInt.
  - totalReceived: The total amount received by this account, represented as a BigInt.
  - sentCount: The number of transfers sent by this account.
    -receivedCount: The number of transfers received by this account.
- Derived Fields:
  - @derivedFrom(field: "from"): This directive indicates that the sentTransfers field should be populated based on the from field of the Transfer entity.
  - @derivedFrom(field: "to"): This directive indicates that the receivedTransfers field should be populated based on the to field of the Transfer entity.

### Step 4: Helper Functions

In src/utils/helper.ts, define helper functions to manage the creation and retrieval of entities:

````ts
import { BigInt, Bytes } from "@graphprotocol/graph-ts"; // Import necessary types from graph-ts
import { Account } from "../../generated/schema"; // Import the Account entity from the generated Uniswap schema

// Helper function to load or create an Account entity
export function getOrCreateAccount(address: Bytes): Account {
  // Attempt to load an existing Account entity from the store using the address
  let account = Account.load(address.toHex());

  // If the Account entity does not exist, create and initialize a new one
  if (account == null) {
    account = new Account(address.toHex()); // Create a new Account entity with the address as ID
    account.totalSent = BigInt.fromI32(0); // Initialize totalSent field to 0
    account.totalReceived = BigInt.fromI32(0); // Initialize totalReceived field to 0
    account.sentCount = 0; // Initialize sentCount field to 0
    account.receivedCount = 0; // Initialize receivedCount field to 0
    account.save(); // Save the newly created Account entity to the store
  }

  // Return the Account entity, either loaded or newly created
  return account as Account;
}```

Helper Functions: These utility functions such as `getOrCreateAccount` are like trusty sidekicks for managing entities within the subgraph's datastore. They are crucial for creating, retrieving, and updating entities, ensuring everything runs smoothly behind the scenes. This ensures entities are correctly instantiated and updated during event processing.
````

### Step 5: Implementing Mapping Functions Logic

````ts
Mapping functions in src/mappings/uniswap.ts are crucial for translating Ethereum smart contract events into GraphQL entities:

import { Transfer as TransferEvent } from "../../generated/Uniswap/Uniswap"; // Import the Transfer event from the Uniswap contract ABI
import { Transfer } from "../../generated/schema"; // Import the Transfer entity from the generated schema
import { getOrCreateAccount } from "../utils/helper"; // Import the helper function to get or create an Account entity

// Function to handle transfer events
export function handleTransfer(event: TransferEvent): void {
  // Load or create the 'from' account involved in the transfer
  let fromAccount = getOrCreateAccount(event.params.from);
  // Load or create the 'to' account involved in the transfer
  let toAccount = getOrCreateAccount(event.params.to);

  // Update the count of transfers sent by the 'from' account
  fromAccount.sentCount += 1;
  // Update the count of transfers received by the 'to' account
  toAccount.receivedCount += 1;

  // Update the total value of tokens sent by the 'from' account
  fromAccount.totalSent = fromAccount.totalSent.plus(event.params.amount);
  // Update the total value of tokens received by the 'to' account
  toAccount.totalReceived = toAccount.totalReceived.plus(event.params.amount);

  // Create a new Transfer entity with the unique transaction hash
  let transfer = new Transfer(event.transaction.hash.toHex());
  // Set the 'from' field of the Transfer entity to the 'from' account ID
  transfer.from = fromAccount.id;
  // Set the 'to' field of the Transfer entity to the 'to' account ID
  transfer.to = toAccount.id;
  // Set the value of the transfer to the value from the event
  transfer.value = event.params.amount;
  // Set the timestamp of the transfer to the block timestamp
  transfer.timestamp = event.block.timestamp;

  // Save the updated 'from' account entity to the store
  fromAccount.save();
  // Save the updated 'to' account entity to the store
  toAccount.save();
  // Save the new Transfer entity to the store
  transfer.save();
}```

Mapping Functions: Functions like handleTransfer act as event handlers for Ethereum smart contract events (TransferEvent). They instantiate new GraphQL entities (Transfer) using event data, update related account entities (fromAccount and toAccount), and ensure these changes are persistently stored in the subgraph's datastore.
````

## Sample Queries

## This query fetches the account with the highest total amount received and sent (totalReceived, totalSent)

```gql
query GetAccountWithHighestSentAndReceived {
  highestReceived: accounts(
    orderBy: totalReceived
    orderDirection: desc
    first: 1
  ) {
    id
    totalReceived
    receivedCount
  }
  highestSent: accounts(orderBy: totalSent, orderDirection: desc, first: 1) {
    id
    totalSent
    sentCount
  }
}
```

### Result-1

```gql
{
  "data": {
    "highestReceived": [
      {
        "id": "0xceb69f6342ece283b2f5c9088ff249b5d0ae66ea",
        "totalReceived": "4346430298149661862912400",
        "receivedCount": 7
      }
    ],
    "highestSent": [
      {
        "id": "0xceb69f6342ece283b2f5c9088ff249b5d0ae66ea",
        "totalSent": "4273064450700601552394449",
        "sentCount": 145
      }
    ]
  }
}
```

After the first query, I discovered the same account had sent and received the highest amount so my spidey senses started tingling (LOL)

I decided to dig a little deeper to see it's last five interactions in transfers and receipts with the highest values and the accounts that sent and received from it (I think I have a bit of OCD)

## This query fetchs the top 5 accounts that a specific account has transacted with and their values

```gql
query GetTopInteractions {
  account(id: "0xceb69f6342ece283b2f5c9088ff249b5d0ae66ea") {
    id
    totalSent
    totalReceived
    sentCount
    receivedCount
    sentTransfers(orderBy: value, orderDirection: desc, first: 5) {
      to {
        id
      }
      value
      timestamp
    }
    receivedTransfers(orderBy: value, orderDirection: desc, first: 5) {
      from {
        id
      }
      value
      timestamp
    }
  }
}
```

### Result-2

```gql
{
  "data": {
    "account": {
      "id": "0xceb69f6342ece283b2f5c9088ff249b5d0ae66ea",
      "totalSent": "4273064450700601552394449",
      "totalReceived": "4346430298149661862912400",
      "sentCount": 145,
      "receivedCount": 7,
      "sentTransfers": [
        {
          "to": {
            "id": "0xdfe696ff5443a42dd4c53be979e9de4ce6e979c6"
          },
          "value": "313984368071616599889300",
          "timestamp": "1719893471"
        },
        {
          "to": {
            "id": "0xe2cf7f78238cf9e4e329d341fb77dcee088f56c5"
          },
          "value": "129316050200000000000000",
          "timestamp": "1719937439"
        },
        {
          "to": {
            "id": "0xfc2bb1e27c5d6aec85b8b6731999041a7aa6ab3e"
          },
          "value": "109707121825329314672250",
          "timestamp": "1720038575"
        },
        {
          "to": {
            "id": "0xfc2bb1e27c5d6aec85b8b6731999041a7aa6ab3e"
          },
          "value": "109707121825329314672250",
          "timestamp": "1720038599"
        },
        {
          "to": {
            "id": "0xa91d546814feb61eff741007f2bf21e244a633a9"
          },
          "value": "109707121825329314672250",
          "timestamp": "1720038551"
        }
      ],
      "receivedTransfers": [
        {
          "from": {
            "id": "0xc5e3e60c107da406540611437b35a04f62acd7e9"
          },
          "value": "1226825308183760000000000",
          "timestamp": "1720034591"
        },
        {
          "from": {
            "id": "0xc5e3e60c107da406540611437b35a04f62acd7e9"
          },
          "value": "1132916709238291000000000",
          "timestamp": "1720030811"
        },
        {
          "from": {
            "id": "0xf74cad18819866f07ba83cdc3b53211b45246129"
          },
          "value": "497172139725000000000000",
          "timestamp": "1719901991"
        },
        {
          "from": {
            "id": "0xa1faf10424969a9d5036def19d38d826f864e40d"
          },
          "value": "413605214944582770910600",
          "timestamp": "1719892955"
        },
        {
          "from": {
            "id": "0x5c739d009f42470ae169cc8a576e1831a228157b"
          },
          "value": "413605214944582770910600",
          "timestamp": "1719945239"
        }
      ]
    }
  }
}
```

I then got a bit more obsessed after noticing it had received only 7 interactions. I decided to dig up the details of these 7 addresses to understand more about their interactions. I think I should stop now before I get sucked in (LOL)

### This will give you the requested information about the 7 accounts from which 0xceb69f6342ece283b2f5c9088ff249b5d0ae66ea received the highest amounts, along with their timestamps

```gql
query GetAccountReceivedFrom {
  account(id: "0xceb69f6342ece283b2f5c9088ff249b5d0ae66ea") {
    receivedTransfers(orderBy: value, orderDirection: desc, first: 7) {
      from {
        id
      }
      value
      timestamp
    }
  }
}
```

### Result-3

After the first query, I discovered the same account had sent and received the highest amount so my spidey senses started tingling (LOL)

```gql
{
  "data": {
    "account": {
      "receivedTransfers": [
        {
          "from": {
            "id": "0xc5e3e60c107da406540611437b35a04f62acd7e9"
          },
          "value": "1226825308183760000000000",
          "timestamp": "1720034591"
        },
        {
          "from": {
            "id": "0xc5e3e60c107da406540611437b35a04f62acd7e9"
          },
          "value": "1132916709238291000000000",
          "timestamp": "1720030811"
        },
        {
          "from": {
            "id": "0xf74cad18819866f07ba83cdc3b53211b45246129"
          },
          "value": "497172139725000000000000",
          "timestamp": "1719901991"
        },
        {
          "from": {
            "id": "0xa1faf10424969a9d5036def19d38d826f864e40d"
          },
          "value": "413605214944582770910600",
          "timestamp": "1719892955"
        },
        {
          "from": {
            "id": "0x5c739d009f42470ae169cc8a576e1831a228157b"
          },
          "value": "413605214944582770910600",
          "timestamp": "1719945239"
        },
        {
          "from": {
            "id": "0x59c31e27700c7bf1338ef05f2b691f20b7cc2a5a"
          },
          "value": "412305711113445321091200",
          "timestamp": "1719892847"
        },
        {
          "from": {
            "id": "0x9478a441115489b1bc33b1c4b1de8bb7fbd315d1"
          },
          "value": "250000000000000000000000",
          "timestamp": "1719851171"
        }
      ]
    }
  }
}
```

What if I became santa and wanted to reward the top 10 lifetime users of my dApp?

### This query provides a snapshot of the top accounts by lifetime transfers, whether they are sending or receiving tokens

```graphql
query TopAccountsLifetime {
  topSenders: accounts(
    where: { totalSent_not: null }
    orderBy: totalSent
    orderDirection: desc
    first: 10
  ) {
    id
    totalSent
    sentCount
  }
  topReceivers: accounts(
    where: { totalReceived_not: null }
    orderBy: totalReceived
    orderDirection: desc
    first: 10
  ) {
    id
    totalReceived
    receivedCount
  }
}
```

## Resul 4

```gql
{
  "data": {
    "topSenders": [
      {
        "id": "0xceb69f6342ece283b2f5c9088ff249b5d0ae66ea",
        "totalSent": "4273064450700601552394449",
        "sentCount": 145
      },
      {
        "id": "0x28c6c06298d514db089934071355e5743bf21d60",
        "totalSent": "3717657948491710000000000",
        "sentCount": 240
      },
      {
        "id": "0xa9d1e08c7793af67e9d92fe308d5697fb81d3e43",
        "totalSent": "3674360203473204967367645",
        "sentCount": 726
      },
      {
        "id": "0x1d42064fc4beb5f8aaf85f4617ae8b3b5b8bd801",
        "totalSent": "3287823831334379735244013",
        "sentCount": 1438
      },
      {
        "id": "0xc5e3e60c107da406540611437b35a04f62acd7e9",
        "totalSent": "2359742017422051000000000",
        "sentCount": 2
      },
      {
        "id": "0x63b53181bdc48a9fbf1d23d461d3cfd82b0abc83",
        "totalSent": "2359742017422051000000000",
        "sentCount": 2
      },
      {
        "id": "0xa69babef1ca67a37ffaf7a485dfff3382056e78c",
        "totalSent": "1515494461063009677945733",
        "sentCount": 369
      },
      {
        "id": "0x51c72848c68a965f66fa7a88855f9f7784502a7f",
        "totalSent": "1444504211000862243110291",
        "sentCount": 1299
      },
      {
        "id": "0xf13def4c901b20310a3a178e02cbd443aae72ac8",
        "totalSent": "1234808748373082214050200",
        "sentCount": 1
      },
      {
        "id": "0xe91df726af7f865256982dac350522617a2a785e",
        "totalSent": "1234808748373082214050200",
        "sentCount": 1
      }
    ],
    "topReceivers": [
      {
        "id": "0xceb69f6342ece283b2f5c9088ff249b5d0ae66ea",
        "totalReceived": "4346430298149661862912400",
        "receivedCount": 7
      },
      {
        "id": "0xa9d1e08c7793af67e9d92fe308d5697fb81d3e43",
        "totalReceived": "3821901549290130357910422",
        "receivedCount": 146
      },
      {
        "id": "0x1d42064fc4beb5f8aaf85f4617ae8b3b5b8bd801",
        "totalReceived": "3381159179699522082032134",
        "receivedCount": 1418
      },
      {
        "id": "0x28c6c06298d514db089934071355e5743bf21d60",
        "totalReceived": "2465354169372016058225875",
        "receivedCount": 433
      },
      {
        "id": "0xc5e3e60c107da406540611437b35a04f62acd7e9",
        "totalReceived": "2359742017422051000000000",
        "receivedCount": 2
      },
      {
        "id": "0xa69babef1ca67a37ffaf7a485dfff3382056e78c",
        "totalReceived": "1515793893005500650268082",
        "receivedCount": 352
      },
      {
        "id": "0x51c72848c68a965f66fa7a88855f9f7784502a7f",
        "totalReceived": "1463369874776041858634747",
        "receivedCount": 1063
      },
      {
        "id": "0x21a31ee1afc51d94c2efccaa2092ad1028285549",
        "totalReceived": "1121493709009500000000000",
        "receivedCount": 5
      },
      {
        "id": "0x4c6007e38ce164ed80ff8ff94192225fcdac68cd",
        "totalReceived": "1092956768992595233427269",
        "receivedCount": 123
      },
      {
        "id": "0x68d3a973e7272eb388022a5c6518d9b2a2e66fbf",
        "totalReceived": "770306497815843179338958",
        "receivedCount": 543
      }
    ]
  }
}
```

Thats a wrap for the first session of the beginner masterclass series! Ready to jump on the next one?

Follow me to stay updated!
