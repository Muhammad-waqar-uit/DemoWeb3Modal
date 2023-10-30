```markdown
# WalletConnect V2

Wallets play a crucial role in Web3 applications by enabling the signing of transactions on the Ethereum blockchain. While developing a dapp, implementing wallet functionality can be cumbersome, and creating a separate wallet specific to your application can be time-consuming.

To provide a consistent user interface and experience for dapps in web browsers and mobile environments, a standardized approach (such as EIP-1193) is necessary. Currently, in desktop environments, MetaMask and various mobile wallets offer compliant implementations of such standards.

[WalletConnect](https://docs.walletconnect.com/2.0/) is an open protocol that connects "wallets" with dapps, allowing dapps to support multiple wallets. In particular, [Web3Modal](https://web3modal.com) is a package that simplifies the integration of multiple wallets in a dapp, built on top of WalletConnect.

WalletConnect plans to discontinue its V1 service after June 2023 [as per this plan](https://medium.com/walletconnect/weve-reset-the-clock-on-the-walletconnect-v1-0-shutdown-now-scheduled-for-june-28-2023-ead2d953b595), and the development of V2 is currently in progress. Some wallets have already adopted V2, and you can check the supported wallets on the [WalletConnect Explorer](https://explorer.walletconnect.com/). Please note that MetaMask Mobile does not yet support V2.

WalletConnect V2 is designed to be "chain agnostic," meaning it can be applied to various blockchains, not limited to Ethereum.

## Web3Modal

In this example, we've created a simple React-based example using Web3Modal V2 (please note that V2 is still in progress and may undergo changes). This example uses the following versions:

- `react": "^18.2.0`
- `@web3modal/ethereum": "^2.2.0`
- `@web3modal/react": "^2.2.0`
- `ethers": "^5.7.2`
- `wagmi": "^0.12.6"

Web3Modal should be used in conjunction with the wagmi library, which is specifically designed for React. The `configureChains` and `createClient` functions provided by wagmi are used to set up chain and wallet "clients."

```javascript
const { provider, webSocketProvider, chains } = configureChains(
    [mainnet, goerli, sepolia],
    [w3mProvider({ projectId: PROJECT_ID })]
);
```

In `configureChains`, you configure the chains to connect to and the provider to use when sending transactions. By default, you can use the provider provided by Web3Modal (`w3mProvider`), but in this example, we configure Alchemy as the provider to use WebSockets. Multiple providers can be configured, and in case of failure, it acts as a fallback.

```javascript
const { provider, webSocketProvider, chains } = configureChains(
    [mainnet, goerli, sepolia],
    [jsonRpcProvider({
        rpc: (chain) => getJsonRpcProviders(chain.id),
    })],
);
```

In `getJsonRpcProviders`, you set up providers for each chain (in this case, mainnet, goerli, and sepolia). Since we are using WebSockets, you can configure it as follows:

```javascript
{
    http: `https://eth-mainnet.g.alchemy.com/v2/${API_KEY_ALCHEMY}`,
    webSocket: `wss://eth-mainnet.g.alchemy.com/v2/${API_KEY_ALCHEMY}`,
}
```

`createClient` is used to create a wallet "client" that can be injected into your dapp. 

```javascript
const wagmiClient = createClient({
    autoConnect: true,
    connectors: [],
    provider,
    webSocketProvider,
});
```

In the current architecture, mobile wallets are used for electronic signatures. To connect a mobile wallet to your dapp, you can either scan the dapp's QR code or directly connect from a mobile browser. The selected wallet app is then launched, and signed transactions are sent to the blockchain through the configured provider.

The key component is the `connectors` array, which represents connectors to various wallets. Web3Modal is designed to work with wallets that support the WalletConnect protocol. You can check the list of WalletConnect-supported wallets on their [website](https://explorer.walletconnect.com/). In this example, we configure it for desktop usage with MetaMask, Coinbase, and Ledger.

```javascript
connectors: [
    ...w3mConnectors({
        projectId: PROJECT_ID,
        version: 2, // WalletConnect V2
        chains,
    }),
    coinbaseConnector,
    ledgerConnector,
]
```

`w3mConnectors` are connectors designed for mobile wallets that support WalletConnect. You must set the `projectId`, which corresponds to an API key that needs to be obtained in advance from the WalletConnect [cloud](https://cloud.walletconnect.com/).

This setup allows users to click on the wallet button, and the mobile wallet app will be launched. Users can scan a QR code on the desktop or connect directly from a mobile browser to link their wallet to the dapp.

The wagmi `createClient`-generated client is then converted back into an EthereumClient from Web3Modal, which is used to control the wallet within the dapp.

```javascript
const ethereumClient = new EthereumClient(wagmiClient, chains);
```

## wagmi React Hook

Since Web3Modal is used in conjunction with wagmi, you can utilize various [wagmi React hooks](https://wagmi.sh/react/getting-started). For example, to access information about the connected account in a function, you can use `useAccount`.

```javascript
const { address, isConnected } = useAccount();
```

You can also set up event handlers for when an account is connected or disconnected.

```javascript
useAccount({
    onConnect({ address }) {
        setAddress(address);
    },
    onDisconnect() {


        setAddress("");
        setAlert("Disconnected");
    },
});
```

wagmi react provides `usePrepareSendTransaction` and `useSendTransaction` hooks for transaction preparation and execution. `usePrepareSendTransaction` allows you to preview a transaction and determine whether it will succeed or fail. It's particularly useful for handling conditions like insufficient balance, ENS lookups, and other checks before the transaction is executed. 

```javascript
const { config } = usePrepareSendTransaction({
    request: TEST_TX_OBJ,
    onError(error) {
        console.log(error.message);
        if (isConnected) setAlert(error.message);
    },
});
```

When there are no errors, you can use the `sendTransaction` function provided by `useSendTransaction` to send the transaction.

```javascript
const { sendTransaction } = useSendTransaction({
    ...config,
    onSuccess(tx) {
        const hash = checkTxHash(tx);
        setTxInfo({ txHash: hash, toAddress: TEST_TX_OBJ.to });
    },
    onError(error) {
        // Handle errors
    },
});
```

You can define actions to take upon a successful transaction or an error. In this example, it monitors whether the transaction is added to a block after being sent and provides notifications accordingly.

The Alchemy WebSocket subscription API, `alchemy_minedTransactions`, informs you when a specific transaction is added to a block. This is why you set up WebSockets in `configureChains`.

```javascript
provider.send("eth_subscribe", ["alchemy_minedTransactions", {"addresses": [{"to": ..., "from": ...}],"includeRemoved": false, "hashesOnly": true}]);
```

Similar hooks like `usePrepareContractWrite` and `useContractWrite` are provided for making contract function calls.

## Running the Example

You can set your Alchemy API key in `src/web3/GetWeb3.js`. This example is set up for the mainnet (be cautious!), Görli, and Sepolia testnets. The contract used in contract calls is called SimpleStorage and is deployed on both Görli and Sepolia testnets.

The mobile wallets used for testing are [Rainbow](https://rainbow.me/) and [Avacus](https://avacus.cc/), both of which support WalletConnect V2 and can be used to connect to the Görli testnet.
```

Feel free to use this translated content as the README for your project. If you need any additional information or have further questions, please let me know!
