---
description: Decrypt live and historical SRC20 Transfer and Approval events with seismic-viem
icon: eye
---

# SRC20 Event Decryption

SRC20 `Transfer` and `Approval` events keep token amounts confidential by storing the amount as AES-GCM ciphertext. `seismic-viem` provides helpers for viewing-key registration, live event watching, and the lower-level primitives needed to decrypt historical logs.

The event itself includes an indexed `encryptKeyHash` (`keccak256(viewingKey)`) and an `encryptedAmount`. Filtering on the key hash lets a client fetch only the copies of events that its viewing key can decrypt.

## Live events with a connected wallet

A shielded wallet client can retrieve its registered AES viewing key from the Directory contract with a signed read and then watch both `Transfer` and `Approval` events.

```typescript
const unwatch = await walletClient.watchSRC20Events({
  address: "0xYourSRC20Token",
  onTransfer: (log) => {
    console.log(`${log.from} -> ${log.to}: ${log.decryptedAmount}`);
  },
  onApproval: (log) => {
    console.log(`${log.owner} -> ${log.spender}: ${log.decryptedAmount}`);
  },
  onError: (error) => console.error("event decryption failed", error),
});

// Stop both subscriptions when they are no longer needed.
unwatch();
```

`watchSRC20Events` throws when the connected address does not have a viewing key registered in the Directory contract.

## Live events with an explicit viewing key

For a service, script, or other flow without the wallet that owns the viewing key, use the public-client action and pass the 32-byte AES key explicitly:

```typescript
import type { Hex } from "viem";

const viewingKey: Hex = "0x...";

const unwatch = await publicClient.watchSRC20EventsWithKey(viewingKey, {
  address: "0xYourSRC20Token",
  onTransfer: (log) => console.log(log.decryptedAmount),
  onApproval: (log) => console.log(log.decryptedAmount),
  onError: (error) => console.error(error),
});
```

Both watchers accept the same callback parameters:

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `address` | `Address` | Yes | SRC20 contract address |
| `onTransfer` | `(log: DecryptedTransferLog) => void` | No | Called for each decrypted `Transfer` |
| `onApproval` | `(log: DecryptedApprovalLog) => void` | No | Called for each decrypted `Approval` |
| `onError` | `(error: Error) => void` | No | Called when a matching log cannot be decrypted |

The returned `Promise<() => void>` resolves to a function that stops both event subscriptions.

## Viewing-key registration and management

The Directory helpers are exported from `seismic-viem`:

```typescript
import {
  checkRegistration,
  computeKeyHash,
  getKey,
  getKeyHash,
  registerKey,
} from "seismic-viem";
```

Check whether an address has registered a key and inspect its public key hash:

```typescript
const registered = await checkRegistration(walletClient, walletClient.account.address);

if (registered) {
  const keyHash = await getKeyHash(walletClient, walletClient.account.address);
  console.log("registered viewing-key hash", keyHash);
}
```

`getKey(walletClient)` performs a signed read of the caller's viewing key. The key itself is confidential; `getKeyHash` reads only the public hash used by event filters.

To register a 32-byte AES viewing key:

```typescript
import type { Hex } from "viem";

const viewingKey: Hex = "0x...";
const transactionHash = await registerKey(walletClient, viewingKey);
```

Keep viewing keys secret. Anyone who obtains a key can decrypt the SRC20 event copies encrypted for that key.

## Decrypted log shape

The watcher callbacks expose the original encrypted fields together with the plaintext amount:

```typescript
type DecryptedTransferLog = {
  from: Address;
  to: Address;
  encryptKeyHash: Hex;
  encryptedAmount: Hex;
  decryptedAmount: bigint;
  transactionHash: Hex;
  blockNumber: bigint;
};

type DecryptedApprovalLog = {
  owner: Address;
  spender: Address;
  encryptKeyHash: Hex;
  encryptedAmount: Hex;
  decryptedAmount: bigint;
  transactionHash: Hex;
  blockNumber: bigint;
};
```

## Historical Transfer events

The live watcher uses viem's contract-event subscription API internally. For historical data, fetch logs with viem, filter them by `encryptKeyHash`, and use the same exported AES helpers to decrypt each matching `encryptedAmount`.

```typescript
import { parseAbiItem, type Hex } from "viem";
import {
  AesGcmCrypto,
  computeKeyHash,
  parseEncryptedData,
} from "seismic-viem";

const viewingKey: Hex = "0x...";
const encryptKeyHash = computeKeyHash(viewingKey);
const aes = new AesGcmCrypto(viewingKey);

const transferEvent = parseAbiItem(
  "event Transfer(address indexed from, address indexed to, bytes32 indexed encryptKeyHash, bytes encryptedAmount)",
);

const logs = await publicClient.getLogs({
  address: "0xYourSRC20Token",
  event: transferEvent,
  args: { encryptKeyHash },
  fromBlock: 0n,
  toBlock: "latest",
});

for (const log of logs) {
  const encryptedAmount = log.args.encryptedAmount;
  if (!encryptedAmount) continue;

  const { ciphertext, nonce } = parseEncryptedData(encryptedAmount);
  const plaintext = await aes.decrypt(ciphertext, nonce);
  const amount = BigInt(plaintext);

  console.log(log.transactionHash, amount);
}
```

Use the corresponding `Approval` signature to retrieve historical approvals:

```typescript
const approvalEvent = parseAbiItem(
  "event Approval(address indexed owner, address indexed spender, bytes32 indexed encryptKeyHash, bytes encryptedAmount)",
);
```

For large block ranges, query in bounded ranges so the RPC provider does not have to return an unbounded log set in one response.

## Why a transaction can emit more than one encrypted event

An SRC20 operation can emit separate encrypted copies for participants that have different registered viewing keys. Those copies represent the same token operation but are encrypted independently for their intended viewers. Filtering by `encryptKeyHash` ensures a watcher receives the event copy associated with its own key.

## Troubleshooting

- **`No AES key registered in Directory for this address`**: register a viewing key before using the wallet action, or use `watchSRC20EventsWithKey` with an explicit key.
- **No matching events**: confirm the viewing key hashes to the `encryptKeyHash` emitted for the account and that the queried block range contains SRC20 activity.
- **Decryption errors**: make sure the full `encryptedAmount` from the log is passed to `parseEncryptedData` and that the matching viewing key is being used.

## See Also

- [SRC20 event watching](src20.md) -- live watcher API reference
- [Shielded Public Client](shielded-public-client.md) -- public client setup
- [Shielded Wallet Client](shielded-wallet-client.md) -- wallet client setup
- [Encrypted Events tutorial](../../../tutorials/src20/encrypted-events.md) -- contract-side event flow
