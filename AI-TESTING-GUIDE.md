# Holokai E2E Testing Guide

## Overview

End-to-end test suite for [Holokai Rental Software](https://stage-app.goholokai.com) built with **Playwright + TypeScript**.

---

## Folder Structure

```
tests/
├── public-reservation/          # Public-facing reservation wizard
│   └── guest-reservation-ai-enhanced.spec.ts
├── dashboard/                   # ARS admin dashboard (requires login)
│   └── dashboard-new-reservation-equipment-only.spec.ts
└── archive/                     # Exploratory / uncertain tests (reference only)

test-config/
├── selectors.ts                 # ★ All known selectors, organized by page
├── test-memory.json             # AI-learned knowledge (auto-updated by tests)
└── test-rules.json              # Business rules and validation config

test-helpers/
├── ai-test-base.ts              # Extended Playwright fixture (smartPage, contextManager)
├── selector-discovery.ts        # Dynamic element discovery
└── email-helper.ts              # Gmail IMAP OTP reader

.claude/commands/                # Claude Code slash commands (skills)
├── run-public-tests.md          → /run-public-tests
├── run-dashboard-tests.md       → /run-dashboard-tests
├── run-all-tests.md             → /run-all-tests
└── add-test.md                  → /add-test

CLAUDE.md                        # Project context loaded automatically by Claude
```

---

## Quick Start

```bash
# Install dependencies
npm install
npx playwright install chromium

# Run dashboard tests (headed)
npm run test:dashboard:headed

# Run public reservation tests (headed)
npm run test:public:headed

# Run all tests
npm test

# View HTML report
npm run test:report
```

---

## Test Account Credentials

| Field | Value |
|-------|-------|
| Login URL | `https://stage-app.goholokai.com/ars/login` |
| Email | `tpolonchuk@unicrew.com` |
| Password | `WKpass1!` |

---

## Writing New Tests

### Which framework to use?

| Test area | Framework |
|-----------|-----------|
| Dashboard (`/ars/**`) | Plain Playwright |
| Public Reservation (`/public-reservation/**`) | AI-Enhanced base OR plain Playwright |

### Dashboard test template

```typescript
import { test, expect } from '@playwright/test';

test.describe('Dashboard — My Feature', () => {
  test('should do something', async ({ page }) => {
    test.setTimeout(60_000);

    await test.step('Login', async () => {
      await page.goto('https://stage-app.goholokai.com/ars/login', { waitUntil: 'domcontentloaded' });
      await page.locator('#loginform-email').fill('tpolonchuk@unicrew.com');
      await page.locator('#loginform-password').fill('WKpass1!');
      await page.locator('#login-form button[type="submit"]').click();
      await page.waitForURL('**/ars**', { timeout: 30_000 });
      console.log('✅ Logged in');
    });

    await test.step('Select London location', async () => {
      await page.waitForTimeout(1500);
      await page.locator('select#nfc_location').selectOption('700');
      await page.waitForTimeout(1500);
      console.log('✅ London selected');
    });

    await test.step('My test step', async () => {
      // your test logic here
    });
  });
});
```

### Public reservation test template

```typescript
import { test, expect } from '@playwright/test';

test.describe('Public Reservation — My Flow', () => {
  test('should complete flow', async ({ page }) => {
    test.setTimeout(120_000);

    await test.step('Navigate to reservation page', async () => {
      await page.goto('https://stage-app.goholokai.com/public-reservation/1');
      await page.waitForTimeout(1500);
    });

    await test.step('Select London location', async () => {
      await page.evaluate(() => {
        const radio = document.querySelector('input[value="700"]') as HTMLInputElement;
        radio?.click();
        radio?.dispatchEvent(new Event('change', { bubbles: true }));
      });
      await page.waitForTimeout(2000);
      console.log('✅ London selected');
    });

    // ... add equipment, time, authorization steps
  });
});
```

---

## Selectors Reference

All selectors are in `test-config/selectors.ts`. Import and use them:

```typescript
import { LOGIN, DASHBOARD, PUBLIC_RESERVATION, equipmentPlusSelector } from '../../test-config/selectors';

// Login
await page.locator(LOGIN.emailInput).fill('...');

// Select London on dashboard
await page.locator(DASHBOARD.locationSelect).selectOption(DASHBOARD.locations.london);

// Click equipment plus button
await page.evaluate((sel) => {
  (document.querySelector(sel) as HTMLButtonElement)?.click();
}, equipmentPlusSelector('Boat'));
```

---

## Selector Stability Notes

| Selector | Stability | Notes |
|----------|-----------|-------|
| `#loginform-email` | Stable | Confirmed working |
| `#loginform-password` | Stable | Confirmed working |
| `select#nfc_location` | Stable | Dashboard location dropdown, id="nfc_location" |
| `input[value="700"]` | Stable | London radio on public reservation page |
| `button.next-button` | Stable | Used across all wizard steps |
| `#startTime` | Stable | Start time dropdown in time step |
| `.equipment` | Stable | Equipment card containers |
| `.quantity-control__button--plus` | Stable | Add equipment button |
| `button.authorization__submit-button` | Stable | Auth form submit |
| `.authorization-verification__digit-input` | Stable | OTP code inputs |
| `a.btn-color.btn-flat` | Use text filter | Both `+ Reservation` and `+ Dynamic checkout` share this class |

---

## Available Claude Code Skills

Use these as slash commands in the Claude Code CLI:

| Command | What it does |
|---------|-------------|
| `/run-public-tests` | Runs `tests/public-reservation/` headed |
| `/run-dashboard-tests` | Runs `tests/dashboard/` headed |
| `/run-all-tests` | Runs all tests headed |
| `/add-test` | Guided flow to create a new test |

---

## Known Gotchas

1. **Post-login URL**: After login, `waitForURL('**/ars**')` — the URL stays at `/ars/login` briefly before redirecting.

2. **Reactive components**: Vue/React components don't always respond to Playwright direct clicks. Use `page.evaluate()` with `.click()` and dispatch `change` events.

3. **Element visibility**: Public reservation equipment items exist in the DOM but may not be "visible" (Vue v-show). Use `{ state: 'attached' }` instead of `{ state: 'visible' }`.

4. **Square payment**: The Square payment iframe is geo-blocked without a US VPN. Tests will log a warning but should not hard-fail on this.

5. **NFC_TEST_MODE banner**: Red warning banner on all staging dashboard pages — this is expected, ignore it in tests.

6. **`+ Reservation` button**: The `<a class="btn btn-color btn-flat">` element has an empty `href` attribute — it's JavaScript-triggered. Click it with Playwright's `.click()` or `page.evaluate()`.

---

## Project Locations

| Location | ID |
|----------|----|
| Berlin | 3 |
| **London** | **700** |
| QA | 5986 |
| Paris | 6051 |
| Amsterdam | 6104 |
| Madrid | 6135 |
| Rome | 6141 |
