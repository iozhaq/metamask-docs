# Ethereum Provider API

MetaMask injects a global API into websites visited by its users at `window.ethereum`.
This API allows websites to request users' Ethereum accounts, read data from blockchains the user is connected to, and suggest that the user sign messages and transactions.
The presence of the provider object indicates an Ethereum user.

```javascript
// this function detects most providers injected at window.ethereum
import detectProvider from '@metamask/detect-provider'

try {
  const provider = await detectProvider()
  // From now on, this should always be true:
  // provider === window.ethereum
  startApp(provider) // initialize your app
} catch (error) {
  console.log('Please install MetaMask!')
}
```

The provider API is specified by [EIP 1193](https://eips.ethereum.org/EIPS/eip-1193), and is designed to be minimal.
We recommend that all dapp developers read the following section, [Basic Usage](#basic-usage).

## Basic Usage

For any non-trivial Ethereum application, or dapp, to work, you will have to:

- Detect the Ethereum provider (`window.ethereum`)
- Detect which Ethereum network the user is connected to
- Get the user's Ethereum account(s)

The snippet at the top of this page is sufficient for detecting the provider.
You can learn how to accomplish the other two by reviewing [this snippet](about:blank).

Although any Ethereum operation can be performed via the provider API, most developers use a convenience library with higher-level abstractions.
The most common such libraries are [ethers](https://www.npmjs.com/package/ethers) and [web3](https://www.npmjs.com/package/web3).
Even if you use a convenience library, you will still need to detect the provider and initialize the library with it.
Other than that, you can generally learn everything you need to know from the documentation of those libraries, without reading this lower-level API.

However, for developers of convenience libraries, and for developers who would like to use features that are not yet supported by their favorite libraries, knowledge of the provider API is essential. Read on for more details.

## Upcoming Provider Changes

**Note:** These changes are _upcoming._ Follow [this GitHub issue](https://github.com/MetaMask/metamask-extension/issues/8077) for details.

If you are new to using the provider, or use `ethers`, you do not have to worry about these changes, and can skip ahead [to the next section](about:blank).

In **Q3 2020**, we are introducing minor breaking changes to this API, which we encourage you to
[read more about here](https://medium.com/metamask/breaking-changes-to-the-metamask-inpage-provider-b4dde069dd0a).

Related specifically to this API, we will:

- Stop emitting `chainIdChanged`, and instead emit `chainChanged`
- Remove the following experimental methods:
  - `ethereum._metamask.isEnabled`
  - `ethereum._metamask.isApproved`

Other changes that may be breaking include:

- We will stop reloading the page on network change

In **Q4 2020**, we will also:

- Stop injecting `web3` into web pages

If you rely on the `window.web3` object currently injected by MetaMask, this _will_ break your dapp.
Please read our [migration guide](about:blank) for more details.

# API

The MetaMask provider API is fully compatible with
[EIP 1193](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1193.md)
and [EIP 1102](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1102.md).

## Properties

### ethereum.isMetaMask

`true` if the user has MetaMask installed, `false` otherwise.

The only non-standard part of the API that is not deprecated.

### ethereum.autoRefreshOnNetworkChange (To Be Removed)

Historically, MetaMask has reloaded dapps if the connected chain (network) changes.
This behavior is controlled via this property, which is by default `true`.
If you wish to disable automatic reloading when the chain changes, set this property to `false` immediately after detecting the Provider:

```javascript
ethereum.autoRefreshOnNetworkChange = false;
```

This behavior will be disabled in **Q3 2020**, at which point this property will be removed.
If your dapp requires a reload when the chain changes, you can re-implement this behavior via this snippet:

```javascript
ethereum.on('chainChanged', () => window.location.reload())
```

## Methods

### ethereum.request(args)

```typescript
interface RequestArguments {
  method: string;
  params?: any;
}

ethereum.request(args: RequestArguments): Promise<unknown>;
```

Use `request` to submit RPC requests to Ethereum via MetaMask.
Returns a `Promise` that resolves to the result of the method call.
The `params` type and the return value will vary by RPC method.
In practice, if a method has any `params`, they are almost always of type `Array<any>`.
If the request fails for any reason, the Promise will reject with an [Ethereum RPC Error](about:blank).

MetaMask supports most standardized Ethereum RPC methods, in addition to a number of methods that may not be
supported by other wallets.
See the MetaMask [RPC API documentation](./rpc-api.html) for details.

#### Example

```javascript
params: [
  {
    from: '0xb60e8dd61c5d32be8058bb8eb970870f07233155',
    to: '0xd46e8dd67c5d32be8058bb8eb970870f07244567',
    gas: '0x76c0', // 30400
    gasPrice: '0x9184e72a000', // 10000000000000
    value: '0x9184e72a', // 2441406250
    data:
      '0xd46e8dd67c5d32be8d46e8dd67c5d32be8058bb8eb970870f072445675058bb8eb970870f072445675',
  },
];

ethereum
  .request({
    method: 'eth_sendTransaction',
    params,
  })
  .then((result) => {
    // The result varies by by RPC method.
    // For example, this method will return a transaction hash hexadecimal string on success.
  })
  .catch((error) => {
    // If the request fails, the Promise will reject with an error.
  });
```

## Events

### ethereum.on(eventName, callback)

The provider supports listening for all events specified in [EIP 1193](https://eips.ethereum.org/EIPS/eip-1193#events).

The following are especially important for managing the state of your dapp:

- `accountsChanged`, returns an array of the currently available accounts.
- `chainChanged`, returns the hex-formatted chain ID string of the currently used chain (formerly referred to as "network").

#### Example

```javascript
ethereum.on('accountsChanged', (accounts) => {
  // Time to reload your interface with accounts[0]!
});

ethereum.on('chainChanged', (chainId) => {
  // Time to make sure any calls are directed to the correct chain!
});
```

#### Chain IDs

Below you will find the chain IDs of the networks MetaMask supports by default.
Consult [chainid.network](https://chainid.network) for more.

| Hex  | Decimal | Network                         |
|------|---------|---------------------------------|
| 0x1  | 1       | Ethereum Main Network (MainNet) |
| 0x3  | 3       | Ropsten Test Network            |
| 0x4  | 4       | Rinkeby Test Network            |
| 0x5  | 5       | Goerli Test Network             |
| 0x2a | 42      | Kovan Test Network              |

### Experimental Methods

## Deprecated API

Historically, the Ethereum provider API has been a mess.
Here, we document the remains of this mess in case you see it used in the wild and wonder what's going on.

_You should **never** rely on any of these methods, properties, or events._

## Deprecated Properties

### ethereum.networkVersion (DEPRECATED)

Returns a numeric string representing the current blockchain's network ID. A few example values:

### ethereum.selectedAddress (DEPRECATED)

Returns a hexadecimal string representing the user's "currently selected" address.

## Deprecated Methods

### ethereum.enable() (DEPRECATED)

Alias for `ethereum.request({ method: 'eth_requestAccounts' })

### `ethereum.sendAsync(payload: Object, callback: Function): void` (DEPRECATED)

```typescript
interface JsonRpcRequest {
  id: string | undefined,
  jsonrpc: '2.0',
  method: string,
  params?: Array<any>,
}

interface JsonRpcResponse {
  id: string | undefined,
  jsonrpc: '2.0',
  method: string,
  result?: unknown,
  error?: Error,
}

ethereum.sendAsync(payload: JsonRpcRequest, callback: Function): void
```

This is the ancestor of `ethereum.request`. It only works for JSON-RPC methods, and takes a JSON-RPC request payload object and an error-first callback function as its arguments.

See [the Ethereum JSON-RPC API](https://eips.ethereum.org/EIPS/eip-1474) for details.

### `ethereum.send(method: string, params: any): Promise<unknown>` (DEPRECATED)

This method is mostly a Promise-based `sendAsync` with `method` and `params` instead of a payload as arguments.
However, it can behave in unexpected ways and should generally be avoided.

## Deprecated Events

### TODO
