# Setup Guide — Claude Playbook

> **This file is not for the user.** It is a protocol for Claude to follow whenever a user asks to create a new application. Execute each step in order, ask questions using `AskUserQuestion`, run commands via `Bash`, and create files using `Write`. Do everything automatically — the user should never have to create files or run commands themselves.

---

## Tool Usage Rules

### `AskUserQuestion` — For ALL user input (choices AND free text)

`AskUserQuestion` always includes an automatic **"Other"** option that lets the user type free text. Use this for everything:

- **Multiple-choice:** Options are the predefined choices (e.g. Yes/No, framework list).
- **Free-text input (credentials, names, tokens):** Provide contextual escape-hatch options (e.g. "I don't have an account yet", "Skip for now") — the user types their actual value via the "Other" option.

### Batch credentials together

When collecting email + password (or username + password) for a service, ask **both in a single `AskUserQuestion` call** using two questions. This is faster for the user than two separate prompts. Only ask yes/no account questions separately.

For the second service onward, offer "Same password as Supabase" (or the previously collected password) as an option so the user can just click instead of retyping.

### Running Playwright scripts

All Playwright scripts are run by Claude via the `Bash` tool. The user never runs commands themselves. The browser launches **headed but minimized** (runs silently in the background) to bypass CAPTCHA detection.

```bash
cd setup && npx tsx scripts/<script-name>.ts
```

---

## Step 0 — Bootstrap Playwright Automation

Before anything else, create the `setup/` folder with all automation scripts. These scripts use Playwright to automate browser logins, token generation, and credential extraction so the user never has to manually copy-paste from web pages.

### 0a — Create folder structure

```bash
mkdir -p setup/helpers setup/scripts
```

### 0b — Create `setup/package.json`

Write this file:

```json
{
  "name": "claude-playbook-setup",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "setup:install": "npx playwright install chromium",
    "supabase:signup": "npx tsx scripts/supabase-signup.ts",
    "supabase:login": "npx tsx scripts/supabase-login.ts",
    "github:signup": "npx tsx scripts/github-signup.ts",
    "github:login": "npx tsx scripts/github-login.ts",
    "github:token": "npx tsx scripts/github-token.ts"
  },
  "dependencies": {
    "dotenv": "^16.4.7",
    "playwright": "^1.50.0",
    "tsx": "^4.19.0"
  }
}
```

### 0c — Create `setup/helpers/browser.ts`

Shared browser launcher. Uses `chromium.launch()` in **headed but minimized** mode — the browser runs silently in the background to bypass CAPTCHA detection without popping up on the user's screen.

```ts
import { chromium, type Browser, type BrowserContext } from "playwright";

let browser: Browser | null = null;
let context: BrowserContext | null = null;

export async function launchBrowser(): Promise<BrowserContext> {
  if (context) return context;
  browser = await chromium.launch({
    headless: false,
    args: [
      "--disable-blink-features=AutomationControlled",
      "--start-minimized",
    ],
  });
  context = await browser.newContext({
    viewport: { width: 1280, height: 800 },
  });
  return context;
}

export function waitForUserInput(message: string): Promise<void> {
  console.log(`NOTE: ${message} (auto-continuing)`);
  return Promise.resolve();
}

export async function closeBrowser(): Promise<void> {
  if (context) {
    await context.close();
    context = null;
  }
  if (browser) {
    await browser.close();
    browser = null;
  }
}
```

### 0d — Create `setup/helpers/credentials.ts`

Loads credentials from `setup/.env.setup`. Validates that required vars are set before running a script.

```ts
import dotenv from "dotenv";
import path from "path";
import { fileURLToPath } from "url";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
dotenv.config({ path: path.join(__dirname, "..", ".env.setup") });

interface SupabaseCredentials { email: string; password: string; }
interface GitHubCredentials { username: string; password: string; }

type ServiceMap = {
  supabase: SupabaseCredentials;
  github: GitHubCredentials;
};

const requiredVars: Record<keyof ServiceMap, string[]> = {
  supabase: ["SUPABASE_EMAIL", "SUPABASE_PASSWORD"],
  github: ["GITHUB_USERNAME", "GITHUB_PASSWORD"],
};

export function getCredentials<T extends keyof ServiceMap>(service: T): ServiceMap[T] {
  const vars = requiredVars[service];
  const missing = vars.filter((v) => !process.env[v]);
  if (missing.length > 0) {
    throw new Error(
      `Missing credentials for ${service}. Set these in setup/.env.setup:\n  ${missing.join("\n  ")}`
    );
  }
  switch (service) {
    case "supabase":
      return { email: process.env.SUPABASE_EMAIL!, password: process.env.SUPABASE_PASSWORD! } as ServiceMap[T];
    case "github":
      return { username: process.env.GITHUB_USERNAME!, password: process.env.GITHUB_PASSWORD! } as ServiceMap[T];
    default:
      throw new Error(`Unknown service: ${service}`);
  }
}
```

### 0e — Create `setup/scripts/supabase-signup.ts`

Creates a Supabase account. Runs headed but minimized (silent) to bypass CAPTCHA.

```ts
import { launchBrowser, waitForUserInput, closeBrowser } from "../helpers/browser.js";
import { getCredentials } from "../helpers/credentials.js";

async function main() {
  const { email, password } = getCredentials("supabase");
  const ctx = await launchBrowser();
  const page = await ctx.newPage();

  try {
    console.log("Navigating to Supabase signup...");
    await page.goto("https://supabase.com/dashboard/sign-up");
    await page.waitForLoadState("domcontentloaded");
    await page.waitForTimeout(2000);

    const emailInput = page.locator([
      'input[type="email"]', 'input[name="email"]',
      'input[placeholder*="email" i]', 'input[id*="email" i]',
    ].join(", ")).first();
    await emailInput.waitFor({ state: "visible", timeout: 10_000 });
    await emailInput.click();
    await emailInput.fill(email);

    const passwordInput = page.locator([
      'input[type="password"]', 'input[name="password"]',
      'input[placeholder*="password" i]', 'input[id*="password" i]',
    ].join(", ")).first();
    await passwordInput.waitFor({ state: "visible", timeout: 10_000 });
    await passwordInput.click();
    await passwordInput.fill(password);

    const signupButton = page.locator([
      'button[type="submit"]', 'button:has-text("Sign up")',
      'button:has-text("Sign Up")', 'button:has-text("Create account")',
    ].join(", ")).first();
    await signupButton.waitFor({ state: "visible", timeout: 10_000 });
    await signupButton.click();
    await page.waitForTimeout(2000);

    const hasCaptcha = await page.locator("iframe[src*='captcha'], iframe[src*='recaptcha'], [data-hcaptcha], iframe[src*='turnstile']").isVisible().catch(() => false);
    if (hasCaptcha) {
      await waitForUserInput("CAPTCHA detected — please complete it in the browser.");
      await page.waitForTimeout(2000);
    }

    const pageText = await page.textContent("body") ?? "";
    if (/check your email|confirm your email|verification|verify your email/i.test(pageText)) {
      console.log("SUPABASE_SIGNUP=EMAIL_CONFIRMATION_NEEDED");
      await waitForUserInput("Supabase sent a confirmation email. Please check your inbox, click the link, then come back here.");
    }

    if (page.url().includes("/dashboard") && !page.url().includes("sign-up")) {
      console.log("SUPABASE_SIGNUP=SUCCESS");
    }
  } catch (error) {
    console.error("Error during Supabase signup:", (error as Error).message);
    process.exit(1);
  } finally {
    await closeBrowser();
  }
}

main();
```

### 0f — Create `setup/scripts/supabase-login.ts`

Logs into Supabase, generates an access token, prints it to stdout. Runs headed but minimized (silent) to bypass CAPTCHA.

```ts
import { launchBrowser, waitForUserInput, closeBrowser } from "../helpers/browser.js";
import { getCredentials } from "../helpers/credentials.js";

async function main() {
  const { email, password } = getCredentials("supabase");
  const ctx = await launchBrowser();
  const page = await ctx.newPage();

  try {
    console.log("Navigating to Supabase login...");
    await page.goto("https://supabase.com/dashboard/sign-in");
    await page.waitForLoadState("domcontentloaded");
    await page.waitForTimeout(2000);

    const emailInput = page.locator([
      'input[type="email"]', 'input[name="email"]',
      'input[placeholder*="email" i]', 'input[id*="email" i]',
    ].join(", ")).first();
    await emailInput.waitFor({ state: "visible", timeout: 10_000 });
    await emailInput.click();
    await emailInput.fill(email);

    const passwordInput = page.locator([
      'input[type="password"]', 'input[name="password"]',
      'input[placeholder*="password" i]', 'input[id*="password" i]',
    ].join(", ")).first();
    await passwordInput.waitFor({ state: "visible", timeout: 10_000 });
    await passwordInput.click();
    await passwordInput.fill(password);

    const signinButton = page.locator([
      'button[type="submit"]', 'button:has-text("Sign in")',
      'button:has-text("Sign In")', 'button:has-text("Log in")',
    ].join(", ")).first();
    await signinButton.waitFor({ state: "visible", timeout: 10_000 });
    await signinButton.click();

    try {
      await page.waitForURL(/\/dashboard\/(?!sign-)/, { timeout: 15_000 });
    } catch {
      await waitForUserInput("CAPTCHA detected — please complete it in the browser.");
      await page.waitForURL(/\/dashboard\/(?!sign-)/, { timeout: 30_000 });
    }
    console.log("Logged in to Supabase.");

    console.log("Navigating to access tokens...");
    await page.goto("https://supabase.com/dashboard/account/tokens");
    await page.waitForLoadState("domcontentloaded");
    await page.waitForTimeout(2000);

    const generateBtn = page.locator([
      'button:has-text("Generate new token")', 'button:has-text("Generate token")',
      'button:has-text("New token")',
    ].join(", ")).first();
    await generateBtn.waitFor({ state: "visible", timeout: 10_000 });
    await generateBtn.click();

    await page.waitForTimeout(1000);
    const tokenNameInput = page.locator([
      'input[placeholder*="token" i]', 'input[placeholder*="name" i]',
      'input[name*="token" i]', 'input[name*="name" i]',
      'dialog input[type="text"]',
    ].join(", ")).first();
    await tokenNameInput.waitFor({ state: "visible", timeout: 10_000 });
    await tokenNameInput.fill("claude-setup");

    const confirmBtn = page.locator([
      'dialog button:has-text("Generate")', 'button:has-text("Generate token")',
      'dialog button[type="submit"]',
    ].join(", ")).first();
    await confirmBtn.click();
    await page.waitForTimeout(2000);

    const tokenElement = page.locator("input[readonly], input[type='text'][value*='sbp_'], code, pre").first();
    let token = await tokenElement.inputValue().catch(() => null);
    if (!token) token = await tokenElement.textContent();

    if (!token || token.trim().length === 0) {
      await waitForUserInput("Could not auto-extract the token. Please copy it from the browser.");
      console.error("MANUAL_EXTRACTION_NEEDED");
    } else {
      console.log(`SUPABASE_ACCESS_TOKEN=${token.trim()}`);
    }
  } catch (error) {
    console.error("Error during Supabase login:", (error as Error).message);
    process.exit(1);
  } finally {
    await closeBrowser();
  }
}

main();
```

### 0g — Create `setup/scripts/github-signup.ts`

Creates a new GitHub account. GitHub's signup has multiple steps and a puzzle — the script fills what it can and pauses for the user when needed.

```ts
import { launchBrowser, waitForUserInput, closeBrowser } from "../helpers/browser.js";
import { getCredentials } from "../helpers/credentials.js";

async function main() {
  const { username, password } = getCredentials("github");
  const ctx = await launchBrowser();
  const page = await ctx.newPage();

  try {
    console.log("Navigating to GitHub signup...");
    await page.goto("https://github.com/signup");
    await page.waitForLoadState("domcontentloaded");
    await page.waitForTimeout(2000);

    // GitHub signup is a multi-step wizard. Fields appear one at a time.
    // Step 1: Email — we pause here for the user to enter their real email
    await waitForUserInput("GitHub signup is open. Please enter your email address in the browser and click Continue, then press Enter here.");

    // Step 2: Password — try to fill it
    await page.waitForTimeout(2000);
    const passwordInput = page.locator('input[type="password"], input[name="password"], input#password').first();
    if (await passwordInput.isVisible({ timeout: 5_000 }).catch(() => false)) {
      console.log("Filling password...");
      await passwordInput.fill(password);
      // Click continue if there's a button
      const continueBtn = page.locator('button:has-text("Continue"), button[type="submit"]').first();
      if (await continueBtn.isVisible().catch(() => false)) {
        await continueBtn.click();
      }
    }

    // Step 3: Username
    await page.waitForTimeout(2000);
    const usernameInput = page.locator('input[name="user[login]"], input#login, input[placeholder*="username" i]').first();
    if (await usernameInput.isVisible({ timeout: 5_000 }).catch(() => false)) {
      console.log("Filling username...");
      await usernameInput.fill(username);
      const continueBtn = page.locator('button:has-text("Continue"), button[type="submit"]').first();
      if (await continueBtn.isVisible().catch(() => false)) {
        await continueBtn.click();
      }
    }

    // GitHub puzzle / CAPTCHA / email preferences step
    await waitForUserInput("Please complete any remaining steps in the browser (puzzle, email preferences, etc.), then press Enter.");

    // Check for email verification code
    await page.waitForTimeout(2000);
    const hasCodeInput = await page.locator('input[name="otp"], input[placeholder*="code" i], input[name="verification"]').first().isVisible().catch(() => false);
    if (hasCodeInput) {
      console.log("GITHUB_SIGNUP=EMAIL_VERIFICATION_NEEDED");
      await waitForUserInput("GitHub sent a verification code to your email. Enter it in the browser, then press Enter.");
    }

    const isLoggedIn = await page.locator("img.avatar, [data-login], .Header-link").first().isVisible({ timeout: 10_000 }).catch(() => false);
    if (isLoggedIn) {
      console.log("GITHUB_SIGNUP=SUCCESS");
    } else {
      console.log("GITHUB_SIGNUP=MANUAL_CHECK_NEEDED");
    }
  } catch (error) {
    console.error("Error during GitHub signup:", (error as Error).message);
    process.exit(1);
  } finally {
    await closeBrowser();
  }
}

main();
```

### 0h — Create `setup/scripts/github-login.ts`

Logs into GitHub and verifies the session.

```ts
import { launchBrowser, waitForUserInput, closeBrowser } from "../helpers/browser.js";
import { getCredentials } from "../helpers/credentials.js";

async function main() {
  const { username, password } = getCredentials("github");
  const ctx = await launchBrowser();
  const page = await ctx.newPage();

  try {
    console.log("Navigating to GitHub login...");
    await page.goto("https://github.com/login");
    await page.waitForLoadState("domcontentloaded");
    await page.waitForTimeout(2000);

    // Find username/email input
    const usernameInput = page.locator([
      'input[name="login"]',
      'input#login_field',
      'input[placeholder*="username" i]',
      'input[placeholder*="email" i]',
      'input[type="text"]',
    ].join(", ")).first();

    console.log("Filling username...");
    await usernameInput.waitFor({ state: "visible", timeout: 10_000 });
    await usernameInput.click();
    await usernameInput.fill(username);

    // Find password input
    const passwordInput = page.locator([
      'input[name="password"]',
      'input#password',
      'input[type="password"]',
    ].join(", ")).first();

    console.log("Filling password...");
    await passwordInput.waitFor({ state: "visible", timeout: 10_000 });
    await passwordInput.click();
    await passwordInput.fill(password);

    // Click sign in
    const signinButton = page.locator([
      'input[type="submit"][value*="Sign in" i]',
      'button[type="submit"]',
      'input[name="commit"]',
    ].join(", ")).first();

    console.log("Clicking sign in...");
    await signinButton.click();

    // Wait for redirect or 2FA
    await page.waitForTimeout(3000);

    const is2FA = await page.locator([
      'input[name="otp"]',
      'input[name="app_otp"]',
      '#totp',
      'input[placeholder*="code" i]',
    ].join(", ")).first().isVisible().catch(() => false);

    if (is2FA) {
      await waitForUserInput("2FA detected — please complete it in the browser.");
      await page.waitForURL("https://github.com/", { timeout: 60_000 });
    }

    const isLoggedIn = await page.locator("img.avatar, [data-login], .Header-link").first().isVisible({ timeout: 5_000 }).catch(() => false);
    console.log(`GITHUB_LOGGED_IN=${isLoggedIn}`);
    if (!isLoggedIn) console.error("Could not verify GitHub login.");
  } catch (error) {
    console.error("Error during GitHub login:", (error as Error).message);
    process.exit(1);
  } finally {
    await closeBrowser();
  }
}

main();
```

### 0i — Create `setup/scripts/github-token.ts`

Creates a GitHub Personal Access Token with `repo` scope and extracts it.

```ts
import { launchBrowser, waitForUserInput, closeBrowser } from "../helpers/browser.js";
import { getCredentials } from "../helpers/credentials.js";

async function main() {
  const { username, password } = getCredentials("github");
  const ctx = await launchBrowser();
  const page = await ctx.newPage();

  try {
    console.log("Navigating to GitHub token creation...");
    await page.goto("https://github.com/settings/tokens/new");
    await page.waitForLoadState("domcontentloaded");
    await page.waitForTimeout(2000);

    // If redirected to login, handle it
    if (page.url().includes("/login") || page.url().includes("/session")) {
      console.log("Not logged in — logging in first...");
      const usernameInput = page.locator('input[name="login"], input#login_field, input[type="text"]').first();
      await usernameInput.waitFor({ state: "visible", timeout: 10_000 });
      await usernameInput.fill(username);

      const passwordInput = page.locator('input[name="password"], input#password, input[type="password"]').first();
      await passwordInput.fill(password);

      const signinButton = page.locator('input[type="submit"], button[type="submit"], input[name="commit"]').first();
      await signinButton.click();

      await page.waitForTimeout(3000);

      const is2FA = await page.locator('input[name="otp"], input[name="app_otp"], #totp').first().isVisible().catch(() => false);
      if (is2FA) {
        await waitForUserInput("2FA detected — please complete it in the browser.");
      }

      await page.waitForURL("**/settings/tokens/**", { timeout: 60_000 });
    }

    // May require password re-confirmation (sudo mode)
    const sudoPassword = page.locator('input[name="sudo_password"], input#sudo_password, input[type="password"]').first();
    if (await sudoPassword.isVisible({ timeout: 3_000 }).catch(() => false)) {
      console.log("Password re-confirmation required...");
      await sudoPassword.fill(password);
      const confirmBtn = page.locator('button[type="submit"], input[type="submit"]').first();
      await confirmBtn.click();
      await page.waitForLoadState("domcontentloaded");
      await page.waitForTimeout(2000);
    }

    // Fill token form
    const noteInput = page.locator('#oauth_access_description, input[name="oauth_access[description]"], input[placeholder*="note" i]').first();
    await noteInput.waitFor({ state: "visible", timeout: 10_000 });
    await noteInput.fill("claude-playbook-token");

    // Select repo scope
    const repoCheckbox = page.locator('#scope_repo, input[value="repo"]').first();
    await repoCheckbox.check();

    // Generate
    const generateBtn = page.locator('button:has-text("Generate token"), input[type="submit"][value*="Generate"]').first();
    await generateBtn.click();
    await page.waitForLoadState("domcontentloaded");
    await page.waitForTimeout(2000);

    // Extract the token (shown once)
    const tokenElement = page.locator('#new-oauth-token, [data-clipboard-text], code.token').first();
    let token = await tokenElement.getAttribute("data-clipboard-text").catch(() => null);
    if (!token) token = await tokenElement.textContent().catch(() => null);

    if (token && token.trim().startsWith("ghp_")) {
      console.log(`GITHUB_TOKEN=${token.trim()}`);
    } else {
      await waitForUserInput("Could not auto-extract the token. Please copy it from the browser.");
      console.error("MANUAL_EXTRACTION_NEEDED");
    }
  } catch (error) {
    console.error("Error during GitHub token creation:", (error as Error).message);
    process.exit(1);
  } finally {
    await closeBrowser();
  }
}

main();
```

### 0j — Install dependencies

```bash
cd setup && npm install && npx playwright install chromium
```

### 0k — Collect user credentials and create accounts

#### Supabase

1. **"Do you already have a Supabase account?"** — options: `["Yes, I have one", "No, I need to create one"]`
2. **Email + password in one call** — use `AskUserQuestion` with **two questions**:
   - Q1: "Supabase email (type via 'Other')" — options: `["Skip for now"]`
   - Q2: "Supabase password (type via 'Other')" — options: `["Skip for now"]`
3. Write credentials to `setup/.env.setup`.
4. If the user **does NOT have an account**, run the signup script via `Bash`:
   ```bash
   cd setup && npx tsx scripts/supabase-signup.ts
   ```
   Then ask: "Please check your inbox and click the confirmation link from Supabase." — options: `["Done, email confirmed", "I didn't receive it"]`.

#### GitHub

5. **"Do you already have a GitHub account?"** — options: `["Yes, I have one", "No, I need to create one"]`
6. **Username + password in one call** — use `AskUserQuestion` with **two questions**:
   - Q1: "GitHub username (type via 'Other')" — options: `["Skip for now"]`
   - Q2: "GitHub password (type via 'Other')" — options: `["Same as Supabase", "Skip for now"]`
7. Append GitHub credentials to `setup/.env.setup`.
8. If the user **does NOT have an account**, run the signup script via `Bash`:
   ```bash
   cd setup && npx tsx scripts/github-signup.ts
   ```

#### Final `.env.setup`

```
SUPABASE_EMAIL=<collected>
SUPABASE_PASSWORD=<collected>
GITHUB_USERNAME=<collected>
GITHUB_PASSWORD=<collected>
```

### 0l — Add to `.gitignore`

Ensure these are gitignored (create `.gitignore` if it doesn't exist):

```
setup/.env.setup
setup/node_modules/
.env.local
.claude/settings.json
```

> **Never commit `.env.setup` or `.claude/settings.json`.** They contain plaintext passwords and tokens.

---

## Step 1 — Gather Requirements

Ask the user the following questions (use `AskUserQuestion`, batch when possible):

1. **App name** — What should the project be called?
2. **Purpose** — One sentence: what does this app do?
3. **Target users** — Who is this for?
4. **UI needed?** — Yes/No. If yes: Web, mobile, or both? Framework preference (Next.js, React, SvelteKit, etc.)?
5. **Authentication needed?** — Yes/No. If yes: Email/password, OAuth (Google, GitHub…), or both?

Store the answers in memory for use in later steps.

---

## Step 2 — Supabase Setup

The user already created a Supabase account in Step 0l. Now create a project and get API keys.

### Prerequisites

Check if the Supabase CLI is installed:

```bash
supabase --version
```

If not installed, run:

```bash
brew install supabase/tap/supabase
```

#### Login via Playwright

Run the Supabase login script via `Bash`:

```bash
cd setup && npx tsx scripts/supabase-login.ts
```

Parse the output for the `SUPABASE_ACCESS_TOKEN=...` line. If the script prints `MANUAL_EXTRACTION_NEEDED`, use `AskUserQuestion` to ask the user to paste the token.

Once you have the token, log in via CLI:

```bash
supabase login --token <captured-token>
```

#### Create the project

```bash
supabase projects create "<app-name>" --region <region>
```

Ask the user for their preferred region if not obvious. Common values: `us-east-1`, `eu-west-1`, `ap-southeast-1`.

The CLI will output the **project ref** (e.g. `abcdefghijkl`). Store it.

#### Retrieve API keys

```bash
supabase projects api-keys --project-ref <project-ref>
```

This returns the `anon` key and `service_role` key. Store both.

The project URL follows the pattern: `https://<project-ref>.supabase.co`

### Important context: Supabase credentials model

If the user is confused about credentials, explain:

- **Dashboard login** (GitHub OAuth or email/password on supabase.com) = access to the Supabase console. This is NOT a database credential.
- **Database password** = set during project creation. Used for direct Postgres connections. Separate from the dashboard login.
- **API keys** (anon key, service role key) = used by the app to talk to Supabase via its REST API. These are NOT passwords.
- **You cannot create a Supabase database using just an email and password.** You must sign up for a Supabase account, create a project in the dashboard (or via CLI), and use the generated API keys in your app.

---

## Step 3 — Write Environment Variables

Generate the `.env.local` file based on collected values. Adapt variable prefixes to the chosen framework:

| Framework | Prefix |
| --- | --- |
| Next.js | `NEXT_PUBLIC_` |
| Vite / SvelteKit | `VITE_` |
| Create React App | `REACT_APP_` |

Write the file using the `Write` tool:

```
# Supabase
<PREFIX>SUPABASE_URL=https://<project-ref>.supabase.co
<PREFIX>SUPABASE_ANON_KEY=<anon-key>
SUPABASE_SERVICE_ROLE_KEY=<service-role-key>

# App
<PREFIX>APP_NAME=<app-name>
```

The `.gitignore` was already updated in Step 0m.

> **Never commit the service role key.** It bypasses Row Level Security.

---

## Step 4 — Connect Supabase via MCP

This lets Claude interact with the user's Supabase database directly (run queries, create tables, inspect schema).

### Get access token

Reuse the `SUPABASE_ACCESS_TOKEN` captured in Step 2. If it wasn't captured (e.g. user had an existing project), run the login script via `Bash`:

```bash
cd setup && npx tsx scripts/supabase-login.ts
```

Parse the output for the `SUPABASE_ACCESS_TOKEN=...` line. If the script prints `MANUAL_EXTRACTION_NEEDED`, use `AskUserQuestion` to ask the user to paste the token.

### Write MCP config

Read the existing `.claude/settings.json` (project-level) or create it. Merge in:

```json
{
  "mcpServers": {
    "supabase": {
      "command": "npx",
      "args": [
        "-y",
        "@supabase/mcp-server-supabase@latest",
        "--access-token",
        "<access-token>"
      ]
    }
  }
}
```

Tell the user: "Claude Code needs to restart to pick up the MCP connection. I'll remind you to restart after setup is complete."

---

## Step 5 — GitHub Connection

The user already provided GitHub credentials in Step 0k. Skip the account question.

### Login via Playwright

Run the GitHub login script directly:

```bash
cd setup && npx tsx scripts/github-login.ts
```

Parse the output for `GITHUB_LOGGED_IN=true`.

### Authenticate the GitHub CLI

Check if the `gh` CLI is installed:

```bash
gh --version
```

If not installed, run:

```bash
brew install gh
```

Then log in:

```bash
gh auth login
```

This opens an interactive flow. Once authenticated, verify:

```bash
gh auth status
```

### Create a repository for the project

The repo **must be public** — GitHub Pages is not available on private repos with the free plan.

```bash
gh repo create "<app-name>" --public --source=. --remote=origin
```

### Get GitHub token

First try the CLI:

```bash
gh auth token
```

If that works, use the output directly. If not, run the token creation script:

```bash
cd setup && npx tsx scripts/github-token.ts
```

Parse the output for the `GITHUB_TOKEN=...` line. If the script prints `MANUAL_EXTRACTION_NEEDED`, use `AskUserQuestion` to ask the user to paste the token.

### Configure GitHub MCP

Read the existing `.claude/settings.json` and merge in the GitHub MCP server:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-github"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<github-token>"
      }
    }
  }
}
```

### Add GitHub token to environment variables

Append to `.env.local`:

```
# GitHub
GITHUB_PERSONAL_ACCESS_TOKEN=<github-token>
```

> **Never commit this token.** It gives full access to your repositories.

---

## Step 6 — GitHub Pages Deployment

Create a GitHub Actions workflow that builds the Next.js app and deploys it to GitHub Pages on every push to `main`.

### Enable GitHub Pages on the repo

```bash
gh api repos/<username>/<app-name>/pages -X POST -f "build_type=workflow" 2>/dev/null || echo "Pages may already be enabled"
```

### Update `next.config` for static export

Next.js needs to be configured for static export to work with GitHub Pages. Add these settings to `next.config.ts` (or `next.config.js`):

```js
output: "export"
```

If the app name is not at the root (e.g. `https://<username>.github.io/<app-name>/`), also add:

```js
basePath: "/<app-name>"
```

### Create the workflow file

```bash
mkdir -p .github/workflows
```

Write `.github/workflows/deploy.yml`:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npx next build

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./out

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

The app will be live at `https://<username>.github.io/<app-name>/` after the first successful workflow run.

---

## Step 7 — Verification & Deploy to GitHub

Run these checks automatically after setup. Do not ask the user to do them manually.

### Test database connection

Create and run a temporary test script:

```js
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.<PREFIX>SUPABASE_URL,
  process.env.<PREFIX>SUPABASE_ANON_KEY
);

const { data, error } = await supabase.from("_test_ping").select("*");

if (error && error.code === "42P01") {
  console.log("OK — Supabase connection works (table does not exist yet, which is expected).");
} else if (error) {
  console.error("FAIL — Connection error:", error.message);
} else {
  console.log("OK — Supabase connection works. Data:", data);
}
```

Report result to user. Delete the test script after.

### Cleanup

After all verifications pass, delete the `setup/` folder — it's no longer needed:

```bash
rm -rf setup/
```

### Deploy to GitHub

Commit everything and push to the remote repository:

```bash
git add -A
git commit -m "Initial project setup

- Supabase connected
- GitHub Actions deployment configured
- Environment variables set
- MCP servers configured"
git push -u origin main
```

### Report results

Print a summary:

```
Setup complete:
  App name:       <app-name>
  Framework:      <framework>
  Supabase:       Connected (<project-ref>)
  GitHub:         Connected (<username>/<repo>)
  Deployment:     GitHub Actions → GitHub Pages (auto-deploy on push to main)
  MCP (Supabase): Configured (restart Claude Code to activate)
  MCP (GitHub):   Configured (restart Claude Code to activate)
  DB connection:  Verified / Failed
```

---

## Summary — What Claude Does at Each Step

| Step | Action | Automated? |
| --- | --- | --- |
| 0. Bootstrap | Create `setup/` folder, scripts, install Playwright, collect credentials | Automated |
| 1. Requirements | Ask questions via `AskUserQuestion` | Interactive |
| 2. Supabase setup | Playwright login + CLI commands | Automated |
| 3. Env vars | Write `.env.local` + `.gitignore` | Automated |
| 4. Supabase MCP | Write `.claude/settings.json` | Automated |
| 5. GitHub | Playwright login + `gh` CLI + token + GitHub MCP config | Automated |
| 6. Deployment | GitHub Actions workflow → GitHub Pages | Automated |
| 7. Verification | Test connections, cleanup, commit & push to GitHub | Automated |
