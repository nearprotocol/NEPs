- Proposal Name: Wallet API
- Start Date: (fill me in with today's date, 2019-07-30)
- NEP PR: [nearprotocol/neps#0000](https://github.com/nearprotocol/NEPs/pull/10)
- Issue(s): n/a

# Summary
[summary]: #summary

We want to have a well-defined API signing transactions to be sent to NEAR protocol via a wallet.
The API needs to support multiple transactions, including but not limited to:
- adding access keys to account
- sending tokens to other accounts
- composite transactions that contain several sub-transactions of potentially different types

In order to support this, we want a generic API for signing arbitrary transactions in the wallet.

# Motivation
[motivation]: #motivation

We want to support multiple wallet implementations, including those by third party developers. We also want to support multiple "styles" of wallets. E.g. web wallets, hardware wallets, browser extensions.
Any specific app may prefer to work with a specific wallet that best suits their use case (security requirements, usability, theme, UX, etc), so there must be a way to configure the app easily to use a particular wallet implementation. We also want a flexible system that allows adding new types of
transactions easily.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Login into app with wallet

App requests wallet to login 
`{walletUrl}/login/?publicKey={publicKey}&contractId={contractId}&callback={callbackUrl}`
- `publicKey` optional, base58 encoded public key to use as app specific key. If not specified – login is only choosing `accountId` and not adding access key
- `walletUrl` is a base wallet URL that can be specified when creating wallet connection using nearlib.
- `contractId` optional, account id of the contract app wants to add access key for. Not needed when `publicKey` is not specified. Means app wants unrestricted access (e.g. `near-shell`) otherwise.
- `callbackUrl` callback URL provided by app that gets opened by wallet after flow completion.

When `publicKey` wallet is expected to complete `addKey` transaction to add access key limited to given `contractId`. User can refuse to do it. This is not an error and will mean that callback only sends `accountId` selected by user back to app.

### Callback URL

Wallet should open `callbackUrl` provided by app specifying following query paramers.

For error:
- `errorCode` unique identifier string for error
  - `requestError` malformed request: missing fields, cannot parse transaction, etc
  - `networkError` user approved transaction but it couldn't be submitted succesfully in allocated time (app still should check later whether it succeeded)
  - `userRejected` user rejected sign in altogether (even not revealing `accountId`)
- `errorDescription` wallet-specific description of error (for debugging purposes only)

For success:
- `accountId` account ID for account used
- `publicKey` optional, public key added as an access key. Not provided if user wants to confirm every transaction explicitly.

## Sign transaction with Web wallet

App requests wallet to sign transaction by open following URL in the browser:
`{walletUrl}/sign?transactions={transactions}&callback={callbackUrl}`

- `walletUrl` is a base wallet URL that can be specified when creating wallet connection using nearlib.
- `transactions` comma-separatted list of base64-encoded [`Transaction` objects](https://github.com/near/near-api-js/blob/db51150b98f3e55c2893a410ad8e2379c10d8b73/src/transaction.ts#L83) serialized using [Borsh](https://borsh.io). 
- `callbackUrl` callback URL provided by app that gets opened by wallet after flow completion.

### Callback URL

Wallet should open `callbackUrl` provided by app specifying following query paramers.

For error:
- `errorCode` unique identifier string for error
  - `requestError` malformed request: missing fields, cannot parse transaction, etc
  - `networkError` user approved transaction but it couldn't be submitted succesfully in allocated time (app still should check later whether it succeeded)
  - `userRejected` user rejected transaction
- `errorDescription` wallet-specific description of error (for debugging purposes only)

For success:
- `transactionHashes` comma-separated list of transaction hashes serialized as base58 strings. 
- `signatures` comma-separated list of base64-encoded [`Signature` objects](https://github.com/near/near-api-js/blob/db51150b98f3e55c2893a410ad8e2379c10d8b73/src/transaction.ts#L78) serialized using [Borsh](https://borsh.io). Order must match `transactions`.

## Caveats

- All unrecognized extra parameters must be ignored (i.e. not trigger `requestError`).
- Wallet need to able to sign multiple transactiosn at one request.

## Alternative integration points

### App-specific URL scheme

URL redirect API described above works for web-based wallets. However it's also applicable with small modification to mobile apps (iOS and Android). Instead of using `https://wallet.host.name/` as `walletUrl` it's possible to register app-specific URL scheme, e.g. `walletname:` and implement everything else in similar way.

### Injected JS

There are some contexts where there might be a way to communicate with wallet continuously vs through URL redirects:
- desktop browser extension
- in-wallet browser on mobile
- web browser with built in wallet
- wallet is available for apps as injectable JS component (e.g. Portis)

In this case communication should happen through `window.nearWalletApi` object defining such methods:

```
async function login({ publicKey, contractId }) { 
    // ...
}

async function sign({ transactions }) { 
    // ...
}
```

Everything else being the same as URL-based inplementation but with such differences:
- parameters are passed as object properties intead of URL query parameters
- "named parameters" using object are utilized (instead of positional) so that it's easy to add new ones later
- instead of taking in `callbackUrl` and redirect promises are used
- promise result should be object with same properties as otherwise would be passed through redirect URL
- in case of error it is thrown with `errorCode` property. `errorDescription` should be used as error message. 

### Examples

```
const { accountId, publicKey } = await nearWalletApi.login({ transactions });
const { transactionHashes, signatures } = await nearWalletApi.sign({ transactions });
```


# Drawbacks
[drawbacks]: #drawbacks

There is limit on transaction size for calls based on URL redirects.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Some apps might prefer "checkout" flow where user doesn't have to login before deciding to send transaction. It is not supported by this proposal.

# Future possibilities
[future-possibilities]: #future-possibilities

- More different integration methods.
- Checkout flow support (i.e. sign partially constructed transaction without preset info about account, nonce, etc)
- More fine-grained error handling.
- Multisig support.
