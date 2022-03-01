<h1 align="center">
  	<img height="180" src="https://github.com/jasonraimondi/promise-watch/raw/master/logos/promise-watch-logo-vertical.png" alt="Promise Watch Logo" />
</h1>

[![Tests](https://github.com/jasonraimondi/promise-watch/actions/workflows/test.yml/badge.svg)](https://github.com/jasonraimondi/promise-watch/actions/workflows/test.yml)
[![License](https://img.shields.io/github/license/jasonraimondi/promise-watch?color=#31C754)](./LICENSE.md)

An Api/E2E monitor that runs promises on intervals and sends notifications on errors. Supports [playwright](https://playwright.dev/) for reliable E2E testing. Has prebuilt [notifiers](#notifiers) for [SMTP](./packages/smtp), [Slack](./packages/slack), and [Pushover](./packages/pushover), and can support any [custom notifier](#custom-notifiers).

Create a `run` directory where you write scripts, set options, then send notifications on errors. Checkout the [example dir](./example) to see a working example.

```
./my-e2e-checks
├── runs
│   ├── checks-https-jasonraimondi-com.ts
│   └── checks-https-google-com.ts
├── src
│   └── main.ts
└── package.json
```

Your runs can be anything! It just needs to export an `options: RunOptions` and `run: Promise<void>`.

```typescript
// runs/checks-https-jasonraimondi-com.ts

import { chromium } from "playwright";
import { RunOptions } from "@promise-watch/core";

export const options: RunOptions = {
  interval: 60, // in seconds
};

export async function run(): Promise<void> {
  const browser = await chromium.launch();
  const page = await browser.newPage();

  const site = "https://jasonraimondi.com";

  const response = await page.goto(site);

  if ((response?.status() ?? 1000) > 399) {
    throw new Error(`${site} Failed to load!`);
  }

  await page.close({ runBeforeUnload: true });
  await browser.close();

  console.log(`success: ${__filename}`);
}
```

## Getting Started

Make a new project directory

```bash
mkdir -p my-watchers/{runs/src}
cd my-watchers
git init
pnpm init -y
```

Install dependencies

```bash
pnpm add @promise-watch/core playwright
pnpm add -D typescript ts-node @types/node
```

Next, create a sample run. A run requires two exports: `options: { interval: number; }` and `run: Promise<void>`. For each 

Add an entrypoint

```typescript
const options: ExecuteOptions = {
  dir: __dirname,
  notifiers: [
    // The ConsoleNotifier logs errors to the console
    // see below for other notifiers
    new ConsoleNotifier(),
  ],
};

executeJobs(options).catch(err => {
  console.error(err);
  process.exit(1);
});
```

And a run script

```json
{
  "scripts": {
    "start": "ts-node src/main.ts"
  }
}
```

And go

```bash
pnpm start
```

## Configuration

The default options:

```typescript
export const options = {
  // required, in seconds
  interval: 30, 
  
  // default: undefined
  notifiers: undefined,
 
  // default: false
  logSuccess: false, 
  
  // default: false
  retryImmediatelyAfterFail: false, 
};
```

## Notifiers

Send notifications when errors occur using the following providers:

* ConsoleNotifier
* [PushoverNotifier](./packages/pushover)
* [SlackNotifier](./packages/slack)
* [SmtpNotifier](./packages/smtp)

```bash
pnpm add @promise-watch/pushover @promise-watch/slack @promise-watch/smtp
```

Then in your execute options, add the `PushoverNotifier` to your `errorNotifiers` array.

```typescript
import { ConsoleNotifier } from "@promise-watch/core";

const options: ExecuteOptions = {
  ...,
  notifiers: [
    new ConsoleNotifier(),
    ...
  ]
}
```

### Custom Notifiers

Implement the Notifier type and you're good to go. See the [pushover notifier](./packages/pushover/src/main.ts) for a working example. Feel free to submit a PR if you want to add support for a custom notifier.

```typescript
export type SendOptions = {
  title: string;
  body: string;
}

export type Notifier = {
  sendError(options: SendOptions): Promise<void>;
  sendRecovered(options: SendOptions): Promise<void>;
}
```

## API Monitoring

Since it is just a Promise with errors being thrown, you can opt to just have a run that just makes an http api request to an endpoint. There is a helper package `@promise-watch/axois` that has a small helper for that.

```typescript
import { checkHttp } from "@promise-watch/axios";

export const options = {
  interval: 30,
}

export async function run() {
  await checkHttp(new URL("https://jasonraimondi.com"));
}
```

## Caveats

For now, this is not going scale to many runs nicely. I'm not sure the limit, but with enough runs, someone will surely find out for us!
