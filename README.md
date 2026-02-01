# Demos Network Agent Registration

Create a Cross-Context Identity (CCI) with GitHub and Twitter linked.

## Quick Start

```bash
node -v && npm list @kynesyslabs/demosdk 2>/dev/null || echo "Run: npm install @kynesyslabs/demosdk"
```

---

## Complete Script

Save as `register.mjs` and run with `npx tsx register.mjs`:

```javascript
import { Demos } from "@kynesyslabs/demosdk/websdk";
import { Identities } from "@kynesyslabs/demosdk/abstraction";
import { execSync } from "child_process";
import fs from "fs";

const sleep = ms => new Promise(r => setTimeout(r, ms));
const CREDS = "demos-identity.json";
const TWITTER_HANDLE = "YOUR_TWITTER_HANDLE"; // CHANGE THIS

async function main() {
  console.log("=== Demos Identity Registration ===\n");

  // 1. CREATE WALLET
  const demos = new Demos();
  await demos.connect("https://demosnode.discus.sh/");
  const mnemonic = demos.newMnemonic();
  await demos.connectWallet(mnemonic);
  const address = demos.getAddress();
  const identities = new Identities();
  const proof = await identities.createWeb2ProofPayload(demos);

  console.log("✓ Wallet created");
  console.log("  Address:", address);

  const creds = { mnemonic, address, proof, created: new Date().toISOString() };
  fs.writeFileSync(CREDS, JSON.stringify(creds, null, 2));

  // 2. CREATE GITHUB GIST
  const ghToken = process.env.GH_TOKEN;
  if (!ghToken) throw new Error("Set GH_TOKEN environment variable");

  const gist = execSync(
    `echo -n '${proof}' | GH_TOKEN="${ghToken}" gh gist create --public -f demos-proof.txt -`,
    { encoding: "utf8" }
  ).trim();

  console.log("✓ Gist created:", gist);
  creds.gist = gist;
  fs.writeFileSync(CREDS, JSON.stringify(creds, null, 2));

  // 3. LINK GITHUB
  const ghResult = await identities.addGithubIdentity(demos, gist);
  const ghBroadcast = await demos.broadcast(ghResult);

  if (!ghBroadcast?.extra?.confirmationBlock) throw new Error("GitHub link failed");
  console.log("✓ GitHub linked at block:", ghBroadcast.extra.confirmationBlock);

  // 4. POST TWEET - Use CLI first (works on headless), fallback to manual
  let tweetUrl = null;

  // Method A: Try ~/tweet CLI (bird) if available
  try {
    console.log("Attempting tweet via CLI...");
    execSync(`~/tweet tweet '${proof}'`, { encoding: "utf8", timeout: 30000 });
    await sleep(5000); // Wait for tweet to propagate

    // Find the tweet URL
    const search = execSync(`~/tweet search 'from:${TWITTER_HANDLE} ${proof.slice(0, 30)}'`, { encoding: "utf8" });
    const urlMatch = search.match(/https:\/\/x\.com\/\w+\/status\/\d+/);
    if (urlMatch) tweetUrl = urlMatch[0];
    console.log("✓ Tweet posted via CLI");
  } catch (e) {
    console.log("CLI tweet failed:", e.message);
  }

  // Method B: Manual fallback
  if (!tweetUrl) {
    console.log("\n⚠️  MANUAL STEP REQUIRED:");
    console.log("Post this tweet manually:");
    console.log(`  ${proof}`);
    console.log("\nThen run: npx tsx link-twitter.mjs <TWEET_URL>");

    creds.twitterPending = true;
    creds.proofToTweet = proof;
    fs.writeFileSync(CREDS, JSON.stringify(creds, null, 2));

    console.log("\n✓ Credentials saved. Complete Twitter linking manually.");
    return;
  }

  console.log("✓ Tweet URL:", tweetUrl);
  creds.tweet = tweetUrl;
  fs.writeFileSync(CREDS, JSON.stringify(creds, null, 2));

  // 5. LINK TWITTER
  const twResult = await identities.addTwitterIdentity(demos, tweetUrl);
  const twBroadcast = await demos.broadcast(twResult);

  if (!twBroadcast?.extra?.confirmationBlock) throw new Error("Twitter link failed");
  console.log("✓ Twitter linked at block:", twBroadcast.extra.confirmationBlock);

  // 6. VERIFY
  const info = await demos.getAddressInfo(address);
  creds.nonce = info?.nonce;
  creds.complete = true;
  fs.writeFileSync(CREDS, JSON.stringify(creds, null, 2));

  console.log("\n=== COMPLETE ===");
  console.log("Address:", address);
  console.log("Nonce:", info?.nonce, "(should be 2)");
  console.log("Credentials:", CREDS);
}

main().catch(e => { console.error("FAILED:", e.message); process.exit(1); });
```

---

## Before Running

1. **Install dependencies** (in your home directory, not /tmp):
   ```bash
   cd ~ && npm install @kynesyslabs/demosdk
   ```

2. **Set environment:**
   ```bash
   export GH_TOKEN="your_github_token"
   ```

3. **Edit the script:** Change `TWITTER_HANDLE` to your Twitter username

4. **For headless servers:** The script uses `~/tweet` CLI (bird) which works without a display. If unavailable, it falls back to manual instructions.

---

## SDK Method Reference

| Task | Exact Call |
|------|------------|
| Create proof | `identities.createWeb2ProofPayload(demos)` |
| Link GitHub | `identities.addGithubIdentity(demos, gistUrl)` |
| Link Twitter | `identities.addTwitterIdentity(demos, tweetUrl)` |
| Commit to chain | `demos.broadcast(result)` ← **REQUIRED after every link** |

---

## Success Indicators

| Step | Success |
|------|---------|
| Wallet | Address starts with `0x`, 66 chars |
| Gist | Returns `https://gist.github.com/...` |
| GitHub link | `broadcast.extra.confirmationBlock` exists |
| Tweet | URL contains `/status/` |
| Twitter link | `broadcast.extra.confirmationBlock` exists |
| Complete | Nonce = 2 |

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| `ERR_UNSUPPORTED_DIR_IMPORT` | Use `npx tsx` not `node` |
| `GH_TOKEN` not set | `export GH_TOKEN="..."` |
| Nonce still 0 | You forgot `demos.broadcast(result)` |
| "Invalid hex string" | Gist has trailing newline - use `echo -n` |
| `Cannot find package` | Run scripts from ~ not /tmp |
| Tweet CLI fails | Falls back to manual - post tweet yourself |

---

## License

MIT - Use freely for AI agent development on Demos Network.
