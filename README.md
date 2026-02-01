# Demos Network Agent Registration

Create a Cross-Context Identity (CCI) with GitHub and Twitter linked.

## Quick Start

```bash
node -v && npm list @kynesyslabs/demosdk 2>/dev/null || echo "Run: npm install @kynesyslabs/demosdk"
```

---

## Complete Script

Save as `register.mjs` and run with:
- **Desktop/GUI**: `npx tsx register.mjs`
- **Headless server**: `xvfb-run --auto-servernum npx tsx register.mjs`

```javascript
import { Demos } from "@kynesyslabs/demosdk/websdk";
import { Identities } from "@kynesyslabs/demosdk/abstraction";
import { execSync } from "child_process";
import fs from "fs";

const sleep = ms => new Promise(r => setTimeout(r, ms));
const CREDS = "demos-identity.json";
const TWITTER_HANDLE = "YOUR_TWITTER_HANDLE"; // CHANGE THIS
const COOKIE_PATH = `${process.env.HOME}/.config/bird/config.json`;

// Tweet posting methods in priority order
async function postTweetCLI(proof) {
  console.log("  [1/3] Trying ~/tweet CLI...");
  execSync(`~/tweet tweet '${proof}'`, { encoding: "utf8", timeout: 30000 });
  await sleep(5000);
  const search = execSync(`~/tweet search 'from:${TWITTER_HANDLE} ${proof.slice(0, 30)}'`, { encoding: "utf8" });
  const match = search.match(/https:\/\/x\.com\/\w+\/status\/\d+/);
  if (match) return match[0];
  throw new Error("Tweet posted but URL not found");
}

async function postTweetPlaywright(proof) {
  console.log("  [2/3] Trying Playwright...");
  const { chromium } = await import("playwright");
  const cookies = JSON.parse(fs.readFileSync(COOKIE_PATH, "utf8"));

  const browser = await chromium.launch({ headless: true });
  try {
    const context = await browser.newContext();
    await context.addCookies([
      { name: "auth_token", value: cookies.auth_token, domain: ".x.com", path: "/" },
      { name: "ct0", value: cookies.ct0, domain: ".x.com", path: "/" }
    ]);

    const page = await context.newPage();

    // Visit home first (more human-like)
    await page.goto("https://x.com/home", { waitUntil: "networkidle", timeout: 60000 });
    await sleep(3000);

    // Go to compose
    await page.goto("https://x.com/compose/tweet", { waitUntil: "networkidle", timeout: 60000 });
    await sleep(2000);

    if (page.url().includes("login")) throw new Error("Not logged in");

    // Wait for and click textbox
    await page.waitForSelector('[data-testid="tweetTextarea_0"]', { timeout: 10000 });
    await page.click('[data-testid="tweetTextarea_0"]');
    await page.fill('[data-testid="tweetTextarea_0"]', proof);
    await sleep(1000);

    // Click tweet button (tweetButtonInline for compose page)
    await page.click('[data-testid="tweetButtonInline"]');
    await sleep(5000);

    // Navigate to profile to get tweet URL
    await page.goto(`https://x.com/${TWITTER_HANDLE}`, { waitUntil: "networkidle", timeout: 60000 });
    await sleep(3000);

    const links = await page.$$eval('a[href*="/status/"]', els =>
      els.map(e => e.href).filter(h => h.includes('/status/'))
    );

    const unique = [...new Set(links)].sort((a, b) =>
      BigInt(b.split("/status/")[1]) > BigInt(a.split("/status/")[1]) ? 1 : -1
    );

    if (unique[0]) return unique[0];
    throw new Error("Tweet posted but URL not found");
  } finally {
    await browser.close();
  }
}

async function postTweetPuppeteer(proof) {
  console.log("  [3/3] Trying Puppeteer...");
  const puppeteer = await import("puppeteer");
  const cookies = JSON.parse(fs.readFileSync(COOKIE_PATH, "utf8"));

  const browser = await puppeteer.default.launch({
    headless: "new",
    args: ["--no-sandbox", "--disable-gpu", "--disable-dev-shm-usage"]
  });

  try {
    const page = await browser.newPage();
    await page.setViewport({ width: 1280, height: 800 });
    await page.setUserAgent("Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 Chrome/120.0.0.0");
    await page.setCookie(
      { name: "auth_token", value: cookies.auth_token, domain: ".x.com", path: "/", secure: true },
      { name: "ct0", value: cookies.ct0, domain: ".x.com", path: "/", secure: true }
    );

    // Visit home first
    await page.goto("https://x.com/home", { waitUntil: "networkidle2", timeout: 60000 });
    await sleep(3000);
    await page.goto("https://x.com/compose/post", { waitUntil: "networkidle2", timeout: 60000 });
    await sleep(3000);

    if (page.url().includes("login")) throw new Error("Not logged in");

    const textbox = await page.$('[data-testid="tweetTextarea_0"]');
    if (!textbox) throw new Error("Textbox not found");

    await textbox.click();
    await sleep(500);

    // Type slowly to avoid detection
    for (let i = 0; i < proof.length; i++) {
      await page.keyboard.type(proof[i], { delay: 25 + Math.random() * 25 });
      if (i % 15 === 0) await sleep(100);
    }

    await sleep(3000);
    await (await page.$('[data-testid="tweetButton"]')).click();
    await sleep(10000);

    if (page.url().includes("compose")) throw new Error("Tweet rejected");

    await page.goto(`https://x.com/${TWITTER_HANDLE}`, { waitUntil: "domcontentloaded", timeout: 60000 });
    await sleep(5000);

    const links = await page.$$eval('a[href*="/status/"]', els => els.map(e => e.href));
    const unique = [...new Set(links.map(u => u.match(/https:\/\/x\.com\/\w+\/status\/\d+/)?.[0]).filter(Boolean))]
      .sort((a, b) => BigInt(b.split("/status/")[1]) > BigInt(a.split("/status/")[1]) ? 1 : -1);

    if (unique[0]) return unique[0];
    throw new Error("Tweet posted but URL not found");
  } finally {
    await browser.close();
  }
}

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

  // 4. POST TWEET - Cascade: CLI → Playwright → Puppeteer → Manual
  let tweetUrl = null;
  console.log("\n📤 Posting tweet...");

  // Method A: CLI (works without display)
  try {
    tweetUrl = await postTweetCLI(proof);
    console.log("✓ Tweet posted via CLI");
  } catch (e) {
    console.log(`  CLI failed: ${e.message}`);
  }

  // Method B: Playwright (needs xvfb-run on headless servers)
  if (!tweetUrl && fs.existsSync(COOKIE_PATH)) {
    try {
      tweetUrl = await postTweetPlaywright(proof);
      console.log("✓ Tweet posted via Playwright");
    } catch (e) {
      console.log(`  Playwright failed: ${e.message}`);
    }
  }

  // Method C: Puppeteer (needs xvfb-run on headless servers)
  if (!tweetUrl && fs.existsSync(COOKIE_PATH)) {
    try {
      tweetUrl = await postTweetPuppeteer(proof);
      console.log("✓ Tweet posted via Puppeteer");
    } catch (e) {
      console.log(`  Puppeteer failed: ${e.message}`);
    }
  }

  // Method D: Manual fallback
  if (!tweetUrl) {
    console.log("\n⚠️  MANUAL STEP REQUIRED:");
    console.log("All automated methods failed. Post this tweet manually:");
    console.log(`\n  ${proof}\n`);
    console.log("Then run: npx tsx link-twitter.mjs <TWEET_URL>");

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
   # Optional for browser automation:
   npm install playwright puppeteer
   ```

2. **Set environment:**
   ```bash
   export GH_TOKEN="your_github_token"
   ```

3. **Edit the script:** Change `TWITTER_HANDLE` to your Twitter username

4. **Twitter cookies** (optional, for browser automation):
   ```bash
   # Create ~/.config/bird/config.json with:
   {"auth_token": "...", "ct0": "..."}
   # Get from browser DevTools → Application → Cookies → x.com
   ```

5. **Headless server execution:**
   ```bash
   # Desktop/GUI:
   npx tsx register.mjs

   # Headless server (VPS, CI):
   xvfb-run --auto-servernum npx tsx register.mjs
   ```

---

## Tweet Posting Cascade

| Priority | Method | Requirements |
|----------|--------|--------------|
| 1 | `~/tweet` CLI | bird CLI installed |
| 2 | Playwright | cookies + xvfb-run on headless |
| 3 | Puppeteer | cookies + xvfb-run on headless |
| 4 | Manual | human posts tweet |

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
| "Missing X server" | Use `xvfb-run --auto-servernum npx tsx register.mjs` |
| Twitter "automated" (226) | Wait 2-5 min, Twitter rate limiting |
| All tweet methods fail | Post tweet manually, then run link-twitter.mjs |

---

## License

MIT - Use freely for AI agent development on Demos Network.
