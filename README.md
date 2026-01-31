# Demos Network: AI Agent Registration Guide

A step-by-step guide for AI agents to create a Demos Network identity with linked blockchain wallets and social accounts.

## What is Demos?

Demos Network provides **Cross-Context Identity (CCI)** - a unified identity that links:
- Blockchain wallets (EVM, Solana, MultiversX, NEAR, TON, XRPL, Aptos)
- Social accounts (Twitter, GitHub, Discord, Telegram)
- Post-quantum cryptographic identities

This allows AI agents to have a verifiable, persistent identity across Web2 and Web3.

---

## Prerequisites

```bash
# Node.js 18+ required
npm install @kynesyslabs/demosdk ethers tweetnacl tweetnacl-util bs58

# IMPORTANT: For Node.js 22+, also install tsx (handles ESM compatibility)
npm install -g tsx
```

**⚠️ Node 22 Compatibility**: The Demos SDK has ESM module resolution issues with Node.js 22+. Use `tsx` instead of `node` to run scripts:
```bash
# Instead of: node register-agent.mjs
# Use:        tsx register-agent.mjs
```

---

## Step 1: Create Your Demos Wallet

```javascript
import { Demos } from "@kynesyslabs/demosdk/websdk";

const demos = new Demos();
await demos.connect("https://demosnode.discus.sh/");

// Generate a 12-word BIP39 mnemonic - SAVE THIS SECURELY!
const mnemonic = demos.newMnemonic();
console.log("Your mnemonic (SAVE THIS!):", mnemonic);

// Connect wallet using mnemonic
await demos.connectWallet(mnemonic);

// Get your Demos address
const demosAddress = demos.getAddress();
console.log("Your Demos Address:", demosAddress);
```

**⚠️ CRITICAL**: Save your mnemonic securely. This is your wallet backup and cannot be recovered if lost.

---

## Step 2: Link an EVM Wallet (Optional)

If you have an Ethereum/EVM wallet you want to link:

```javascript
import { Identities } from "@kynesyslabs/demosdk/abstraction";
import { ethers } from "ethers";

const identities = new Identities();

// Your existing ETH wallet credentials
const ETH_ADDRESS = "0xYourEthAddress";
const ETH_PRIVATE_KEY = "0xYourPrivateKey";

// Create signature proving ownership
const ethWallet = new ethers.Wallet(ETH_PRIVATE_KEY);
const message = `Link to Demos: ${demosAddress}`;
const signature = await ethWallet.signMessage(message);

// Submit to Demos
const xmPayload = {
  method: "inferIdentityFromSignature",
  target_identity: {
    chain: "evm",
    subchain: "mainnet",  // or: sepolia, polygon, arbitrum, optimism, base
    chainId: 1,
    isEVM: true,
    targetAddress: ETH_ADDRESS,
    signature: signature,
    signedData: message
  }
};

const result = await identities.inferXmIdentity(demos, xmPayload);
// IMPORTANT: SDK bug - must broadcast manually!
const broadcastResult = await demos.broadcast(result);
console.log("EVM wallet linked:", broadcastResult);
```

---

## Step 3: Link a Solana Wallet (Optional)

```javascript
import { Keypair } from "@solana/web3.js";
import nacl from "tweetnacl";
import { encodeBase64 } from "tweetnacl-util";

// Your Solana keypair
const solanaKeypair = Keypair.generate(); // or load existing
const solanaAddress = solanaKeypair.publicKey.toBase58();

// Create signature proving ownership
const message = `Link to Demos: ${demosAddress}`;
const messageBytes = new TextEncoder().encode(message);
const signatureBytes = nacl.sign.detached(messageBytes, solanaKeypair.secretKey);

// IMPORTANT: Solana signatures must be base64 encoded!
const signature = encodeBase64(signatureBytes);

const xmPayload = {
  method: "inferIdentityFromSignature",
  target_identity: {
    chain: "solana",
    subchain: "mainnet",  // or: devnet
    isEVM: false,
    targetAddress: solanaAddress,
    signature: signature,
    signedData: message
  }
};

const result = await identities.inferXmIdentity(demos, xmPayload);
const broadcastResult = await demos.broadcast(result);
console.log("Solana wallet linked:", broadcastResult);
```

---

## Step 4: Get Your Web2 Proof Payload

This payload will be used to prove ownership of your social accounts:

```javascript
const identities = new Identities();
const proofPayload = await identities.createWeb2ProofPayload(demos);
console.log("Your proof payload:", proofPayload);
// Output: demos:dw2p:ed25519:0x...
```

Save this payload - you'll need it for Twitter and GitHub linking.

---

## Step 5: Link Twitter

1. **Post a tweet** containing your exact proof payload
2. **Link the tweet to your Demos identity:**

```javascript
const tweetUrl = "https://x.com/YourHandle/status/1234567890";
const result = await identities.addTwitterIdentity(demos, tweetUrl);
// IMPORTANT: SDK bug - must broadcast manually!
const broadcastResult = await demos.broadcast(result);
console.log("Twitter linked:", broadcastResult);
```

---

## Step 6: Link GitHub

1. **Create a public Gist** containing your proof payload:
   - Go to https://gist.github.com
   - Create a new **public** gist
   - Filename: `demos-proof.txt`
   - Content: Your proof payload (e.g., `demos:dw2p:ed25519:0x...`)
   - **IMPORTANT**: No trailing newline! The content must be exactly the proof payload.

2. **Link the Gist to your Demos identity:**

```javascript
const gistUrl = "https://gist.github.com/YourUsername/gist_id";
const result = await identities.addGithubIdentity(demos, gistUrl);
// IMPORTANT: SDK bug - must broadcast manually!
const broadcastResult = await demos.broadcast(result);
console.log("GitHub linked:", broadcastResult);
```

---

## Step 7: Verify Your Identities

```javascript
const allIdentities = await identities.getIdentities(demos);
console.log("Your linked identities:", JSON.stringify(allIdentities?.data?.response, null, 2));
```

Expected output structure:
```json
{
  "xm": {
    "evm": { "mainnet": [{ "address": "0x..." }] },
    "solana": { "mainnet": [{ "address": "..." }] }
  },
  "web2": {
    "twitter": [{ "username": "YourHandle" }],
    "github": [{ "username": "YourUsername" }]
  }
}
```

---

## Complete Registration Script

Save as `register-agent.mjs`:

```javascript
import { Demos } from "@kynesyslabs/demosdk/websdk";
import { Identities } from "@kynesyslabs/demosdk/abstraction";
import fs from "fs";

async function registerAgent() {
  // 1. Connect to Demos
  const demos = new Demos();
  await demos.connect("https://demosnode.discus.sh/");

  // 2. Generate mnemonic and connect wallet
  const mnemonic = demos.newMnemonic();
  await demos.connectWallet(mnemonic);
  const demosAddress = demos.getAddress();

  console.log("=== YOUR NEW DEMOS IDENTITY ===");
  console.log("Mnemonic (SAVE SECURELY!):", mnemonic);
  console.log("Demos Address:", demosAddress);

  // 3. Get Web2 proof payload
  const identities = new Identities();
  const proofPayload = await identities.createWeb2ProofPayload(demos);

  console.log("\n=== WEB2 VERIFICATION ===");
  console.log("Post this exact text as a tweet AND in a GitHub Gist:");
  console.log(proofPayload);

  // 4. Save credentials locally
  const credentials = {
    mnemonic,
    demosAddress,
    proofPayload,
    createdAt: new Date().toISOString(),
    note: "KEEP THIS FILE SECURE - DO NOT SHARE!"
  };

  fs.writeFileSync("my-demos-credentials.json", JSON.stringify(credentials, null, 2));
  console.log("\n✓ Credentials saved to my-demos-credentials.json");

  console.log("\n=== NEXT STEPS ===");
  console.log("1. Tweet the proof payload from your Twitter account");
  console.log("2. Create a public GitHub Gist with the proof payload (no trailing newline!)");
  console.log("3. Run the linking scripts with your tweet URL and Gist URL");
}

registerAgent().catch(console.error);
```

---

## Supported Chains

| Chain | Subchains | Signature Format |
|-------|-----------|------------------|
| `evm` | mainnet, sepolia, polygon, arbitrum, optimism, base | Hex (0x-prefixed) |
| `solana` | mainnet, devnet | **Base64** |
| `multiversx` | mainnet, devnet | Hex |
| `near` | mainnet, testnet | Base58 |
| `ton` | mainnet, testnet | Base64 |
| `xrpl` | mainnet, testnet | Hex |
| `aptos` | mainnet, testnet | Hex |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `ERR_UNSUPPORTED_DIR_IMPORT` on Node 22+ | Use `tsx` instead of `node`: `npm install -g tsx && tsx script.mjs` |
| `inferXmIdentity()` succeeds but nonce doesn't change | **SDK bug**: Call `demos.broadcast(result)` after the SDK method returns |
| Solana "invalid encoding" error | Solana signatures must be **base64** encoded, not hex or bs58 |
| GitHub "Invalid hex string" error | Gist content has a trailing newline - recreate without it |
| Twitter "automated request" error | Wait 1-2 minutes and retry |
| Twitter "daily limit" error | Wait until the next day |
| EVM link fails | Verify signature matches exact message format: `Link to Demos: {address}` |
| `Cannot find package` when running from /tmp | Scripts must run from the directory where SDK is installed (usually project root or ~) |

---

## Security Best Practices

1. **Never share your mnemonic** - It controls your entire identity
2. **Store credentials securely** - Use encrypted storage or a secrets manager
3. **Use separate wallets** - Don't link wallets with significant funds
4. **Verify before signing** - Always check what you're signing

---

## Resources

- **Demos Network**: https://demos.sh
- **SDK Documentation**: https://docs.kynesys.xyz
- **Faucet** (testnet tokens): https://faucet.demos.sh

---

## Example: Checking Your Identity Status

```javascript
import { Demos } from "@kynesyslabs/demosdk/websdk";

const demos = new Demos();
await demos.connect("https://demosnode.discus.sh/");
await demos.connectWallet("your twelve word mnemonic phrase here");

const address = demos.getAddress();
const info = await demos.getAddressInfo(address);

console.log("Address:", address);
console.log("Nonce:", info?.nonce);  // Number of identity transactions
console.log("Balance:", info?.balance);
```

---

*This guide is for AI agents registering on the Demos Network. For human users, see the official documentation at docs.kynesys.xyz.*
