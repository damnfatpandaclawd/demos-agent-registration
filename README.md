# Demos Network Agent Registration

Create a Cross-Context Identity (CCI) with GitHub and Twitter linked.

## Quick Start

**One-liner to check if you can run this:**
```bash
node -v && npm list @kynesyslabs/demosdk 2>/dev/null || echo "Run: npm install @kynesyslabs/demosdk puppeteer"
```

---

## Complete Script

Save as `register.mjs` and run with `npx tsx register.mjs`:

```javascript
import { Demos } from "@kynesyslabs/demosdk/websdk";
import { Identities } from "@kynesyslabs/demosdk/abstraction";
import puppeteer from "puppeteer";
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

  // 3. LINK GITHUB - Method: addGithubIdentity(demos, gistUrl)
  const ghResult = await identities.addGithubIdentity(demos, gist);
  const ghBroadcast = await demos.broadcast(ghResult);

  if (!ghBroadcast?.extra?.confirmationBlock) throw new Error("GitHub link failed");
  console.log("✓ GitHub linked at block:", ghBroadcast.extra.confirmationBlock);

  // 4. POST TWEET WITH PUPPETEER
  const cookiePath = `${process.env.HOME}/.config/bird/config.json`;
  if (!fs.existsSync(cookiePath)) throw new Error("Twitter cookies not found at " + cookiePath);
  const cookies = JSON.parse(fs.readFileSync(cookiePath, "utf8"));

  const browser = await puppeteer.launch({
    headless: true,
    args: ["--no-sandbox", "--disable-gpu", "--disable-dev-shm-usage",
           "--disable-blink-features=AutomationControlled"]
  });

  const page = await browser.newPage();
  await page.setUserAgent("Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 Chrome/120.0.0.0");
  await page.setViewport({ width: 1280, height: 800 });
  await page.setCookie(
    { name: "auth_token", value: cookies.auth_token, domain: ".x.com", path: "/", secure: true },
    { name: "ct0", value: cookies.ct0, domain: ".x.com", path: "/", secure: true }
  );

  // Human-like: visit home first
  await page.goto("https://x.com/home", { waitUntil: "networkidle2", timeout: 60000 });
  await sleep(3000);
  await page.goto("https://x.com/compose/post", { waitUntil: "networkidle2", timeout: 60000 });
  await sleep(3000);

  if (page.url().includes("login")) throw new Error("Not logged into Twitter");

  const textbox = await page.$('[data-testid="tweetTextarea_0"]');
  if (!textbox) throw new Error("Tweet textbox not found");

  await textbox.click();
  await sleep(500);

  // Type SLOWLY - fast typing triggers rejection
  for (let i = 0; i < proof.length; i++) {
    await page.keyboard.type(proof[i], { delay: 25 + Math.random() * 25 });
    if (i % 15 === 0) await sleep(100);
  }

  await sleep(3000);
  await (await page.$('[data-testid="tweetButton"]')).click();
  await sleep(10000);

  if (page.url().includes("compose")) throw new Error("Tweet was rejected");
  console.log("✓ Tweet posted");

  // Get tweet URL
  await page.goto(`https://x.com/${TWITTER_HANDLE}`, { waitUntil: "domcontentloaded", timeout: 60000 });
  await sleep(5000);

  const links = await page.$$eval('a[href*="/status/"]', els =>
    els.map(e => e.href).filter(h => h.includes(`/${TWITTER_HANDLE}/status/`))
  );
  const urls = [...new Set(links.map(u => u.match(/https:\/\/x\.com\/\w+\/status\/\d+/)?.[0]).filter(Boolean))];
  urls.sort((a, b) => BigInt(b.split("/status/")[1]) > BigInt(a.split("/status/")[1]) ? 1 : -1);

  await browser.close();

  const tweetUrl = urls[0];
  if (!tweetUrl) throw new Error("Could not find tweet URL");
  console.log("✓ Tweet URL:", tweetUrl);
  creds.tweet = tweetUrl;
  fs.writeFileSync(CREDS, JSON.stringify(creds, null, 2));

  // 5. LINK TWITTER - Method: addTwitterIdentity(demos, tweetUrl)
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
   cd ~ && npm install @kynesyslabs/demosdk puppeteer
   ```

2. **Set environment:**
   ```bash
   export GH_TOKEN="your_github_token"
   ```

3. **Twitter cookies** must exist at `~/.config/bird/config.json`:
   ```json
   {"auth_token": "...", "ct0": "..."}
   ```
   Get from browser DevTools → Application → Cookies → x.com

4. **Edit the script:** Change `TWITTER_HANDLE` to your Twitter username

5. **For headless servers (VPS/CI):** The script uses `headless: true` by default. If you see "Missing X server" errors, ensure you're using Puppeteer (not Playwright) and `headless: true` is set.

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
| Tweet | Redirects away from `/compose/post` |
| Twitter link | `broadcast.extra.confirmationBlock` exists |
| Complete | Nonce = 2 |

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| `ERR_UNSUPPORTED_DIR_IMPORT` | Use `npx tsx` not `node` |
| `GH_TOKEN` not set | `export GH_TOKEN="..."` |
| Twitter cookies not found | Create `~/.config/bird/config.json` |
| Tweet rejected | Puppeteer types too fast - already fixed in script |
| Nonce still 0 | You forgot `demos.broadcast(result)` |
| "Invalid hex string" | Gist has trailing newline - use `echo -n` |
| "Missing X server" or "XServer running" | Ensure `headless: true` is set in Puppeteer launch options |
| Puppeteer not found | Run `npm install puppeteer` in your home directory |

---

## License

MIT - Use freely for AI agent development on Demos Network.
