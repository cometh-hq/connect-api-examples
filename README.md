# How It Works

The objective of this documentation is to familiarize you with Cometh Connect, with a particular focus on its API. By the end of this tutorial, you'll have the skills to create a wallet and execute transactions seamlessly through our API.

**Note:** Don't forget to explore our Connect SDK, which encapsulates most of the logic discussed here.

To begin, you'll need an API key, as it's a mandatory requirement for any interaction with the Connect API. To obtain your API key, please create a project in the Cometh Dashboard. By default, this example uses chainId 80001 (Polygon Mumbai), so when setting up your project, please select this network. 

## General Info

Here are some of the parameters we will use:
- API URL: https://api.connect.cometh.io/
- ChainId: 80001

The chainId is linked to your API key; one project can only be on one chain. If you need to use multiple chains, please use different API keys.



**Note:** You can approach this tutorial in two ways: either by running and reviewing the TypeScript file located at scripts/deploy.ts, or by following the instructions provided in this README and executing the curl commands listed herein. Both methods implement a similar logic: obtaining your project's parameters, creating a wallet, and executing a transaction.

# Use generateWallet script

## Installation

```bash
yarn install
```

## Running

1. Set your API Key in `.env`.
2. Deploy a wallet using:

```bash
yarn generateWallet
```

The result logs should look similar to:
```
Owner address (EOA): 0x240011779626EeCE7c899cb02f017530A32Dd411
Wallet we are about to create: 0x058aB302ED2F8776614aaF6948652BA29F88e45f
Deployment transaction sent to relayer with relayId 0xcdc61283c228081dd1bea5e34ba95fc3a77e5e8f7acd29155e80f4f428b8beae
```

Please wait for a moment until the transaction is confirmed. Once confirmed, you should be able to see your wallet in the blockchain explorer.

**Note:** The relayId is not your transaction ID, so attempting to search for it in the explorer won't yield any results. Instead, search for your wallet address, and you should find the transaction that created it.


# Use curl queries

## Get Project Parameters (Optional)

To ensure that your project parameters are correctly set up, we will perform this step once to verify the validity of your API key and confirm that the chainId of our project matches the blockchain you intend to use.

**Request:**
```bash
curl -X GET -H "apikey: YOUR_API_KEY" https://api.connect.cometh.io/project/params
```

**Response:**
```json
{
  "success": true,
  "projectParams": {
    "chainId": 80001,
    "P256FactoryContractAddress": "0x9Ac319aB147b4f27950676Da741D6184cc305894",
    "multisendContractAddress": "0xA238CBeb142c10Ef7Ad8442C6D1f9E89e07e7761"
  }
}
```

Now that we've confirmed everything is properly configured, let's proceed to create your first wallet.

## Create a Wallet

Next, we'll create a wallet. To achieve this, you only need to supply an owner address, which will serve as the initial owner of the wallet. Keep in mind that, as the wallet is a Safe, you can always add more owners or modify them later on.

**Request:**
```bash
curl -X POST -H "apikey: YOUR_API_KEY" -H "Content-Type: application/json" -d '{"ownerAddress": "YOUR_EOA_ADDRESS"}' https://api.connect.cometh.io/wallets/init
```

**Response:**
```json
{
  "success": true,
  "walletAddress": "0x9E592E4A2aA1A1299AD5cF5fC64d8a0722206c37"
}
```

In this step, it's important to note that we won't create the wallet on-chain just yet. Instead, we will predict the future wallet address. The actual deployment of the wallet will take place in the next step, within the same transaction as our first use of the wallet.

## Use Your Wallet

Our first action will be, as an example, to interact with the function `count()` of a Counter contract, to increment a counter by 1 for our wallet address. To do so, we will first create the `safeTxData` (the data of our action), then sign it, and finally send it to Connect API.

### Prepare the Transaction

Here is the format of a transaction when you want to use your wallet:

```javascript
const safeTxData = {
   to: '0x3633a1be570fbd902d10ac6add65bb11fc914624',
   value: '0',
   data: ethers.utils.id('count()').slice(0, 10),
   operation: '0',
   safeTxGas: '0',
   baseGas: '0',
   gasPrice: '0',
   gasToken: '0x0000000000000000000000000000000000000000',
   refundReceiver: '0x0000000000000000000000000000000000000000',
   nonce: '0'
 }
```

To make this example simple, we consider that the counter contract is sponsored, and so all the gas-related fields can be set to 0 (safeTxGas, baseGas, gasPrice, gasToken, and refundReceiver). We also use a value of 0 since we only intend to interact with the contract, not transfer value (anyway, our wallet does not hold any token at this point).

So the only values we are interested in are:
- `to`: the recipient of our action, here the Counter contract
- `data`: the data for the action we want to do, here simply calling the `count()` function
- `nonce`: the nonce of our wallet. This is the first action of our wallet, so we know it is currently 0. Later on, once the wallet is deployed, it contains a `nonce()` function that can be called to get the current nonce.

### Sign the Transaction

For signing the transaction, we will utilize the `signTypedData` function with our signer. This function takes three parameters:

- `domain`: This includes the `chainId` and `verifyingContract` (which is our wallet address).
- `types`: A constant from EIP 712.
- `value`: The `safeTxData` that we prepared earlier.

```javascript
const signatures = await signer._signTypedData(
    {
      chainId: 80001,
      verifyingContract: walletAddress
    },
    {
      SafeTx: [
        { type: 'address', name: 'to' },
        { type: 'uint256', name: 'value' },
        { type: 'bytes', name: 'data' },
        { type: 'uint8', name: 'operation' },
        { type: 'uint256', name: 'safeTxGas' },
        { type: 'uint256', name: 'baseGas' },
        { type: 'uint256', name: 'gasPrice' },
        { type: 'address', name: 'gasToken' },
        { type: 'address', name: 'refundReceiver' },
        { type: 'uint256', name: 'nonce' }
      ]
    },
    safeTxData
  )
```

The field "signatures" can then be added to the `safeTxData`, which is now ready to be sent. Please note that here we only have one signer, but it's important to keep in mind that a wallet could potentially require multiple signatures.

```javascript
const dataToSend = { ...safeTxData, signatures }
```

### Send the Transaction

With everything set up, you can now send the transaction using the following request. Simply include your API Key in the header, the signature generated in the previous step in the request body, and your wallet address in the request parameters.

**Note:** You can easily acquire a new wallet address and its associated signature by executing the command below:

```bash
yarn getSignature
```

**Request:**
```bash
curl -X POST -H "apikey: YOUR_API_KEY" -H "Content-Type: application/json" -d '{"to": "0x3633a1be570fbd902d10ac6add65bb11fc914624","value": "0","data": "0x06661abd","operation": "0","safeTxGas": "0","baseGas": "0","gasPrice": "0","gasToken": "0x0000000000000000000000000000000000000000","refundReceiver": "0x0000000000000000000000000000000000000000","nonce": "0","signatures": "YOUR_SIGNATURES"}' https://api.connect.cometh.io/wallets/YOUR_WALLET_ADDRESS/relay
```

**Response:**
```json
{
  "success": true,
  "safeTxHash": "0x_YOUR_RELAYER_TX_HASH"
}
```

It's worth noting that the `safeTxHash` is not the same as the transaction hash you'll find on the blockchain. Instead, it serves as an identifier used by the relayer to track and manage your transaction.