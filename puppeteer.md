# Puppeteer

### Firehose by Eric Bidelman:
### https://www.linkedin.com/in/ericbidelman/
* https://devwebfeed.appspot.com/
* https://devwebfeed.appspot.com/?tweets


# Puppeteer

* https://developers.google.com/web/tools/puppeteer
* https://www.youtube.com/watch?v=lhZOFUY1weo
    * t=680
    * https://youtu.be/lhZOFUY1weo?t=680
        * `await page.goto(url, { waitUntil: 'networkidle0' });`
        * `await page.content();` returns searialized HTML content
        * `await page.$eval('html', e => e.outerHTML)`
          * same thing as above
          * may be the html isn't searialized
          * The function page.$eval() must receive two arguments
            * See the method signature: `page.$eval(selector, pageFunction[, ...args])`
              * `selector` (string) A selector to query page for
              * `pageFunction` (function(Element)) Function to be evaluated in browser context
              * `...args` (...Serializable|JSHandle) Arguments to pass to `pageFunction`
              * returns: (Promise(Serializable)) Promise which resolves to the return value of `pageFunction`
    * https://youtu.be/lhZOFUY1weo?t=760
        * Create a server side pre-render module
        * This module shoudl return the HTML content of the page
        * In your `express` server's `get()` method, call the SSR
        * Serve the pre-rendered page as the response
            * `return res.status(200).send(html);` 

### Sample Codes:
#### Example where we get the HTML content of the page:
```js
try {
  const browser = await puppeteer.launch({
    headless: true
  });
  let page = await browser.newPage();
  await page.setViewport({
    width: 1338,
    height: 768,
    deviceScaleFactor: 3
  });
  // networkidle0 waits for the network to be idle (no requests for 500ms).
  // The page's JS has likely produced markup by this point, but wait longer
  // if your site lazy loads, etc.
  await page.goto(url, { waitUntil: 'domcontentloaded' }); //domcontentloaded | networkidle0
  await page.waitForSelector('body');
  let html = await page.$eval('html', e => e.outerHTML);
  await browser.close();
}
catch(error) {
  console.log(error);
}
```


#### SSR Example 0:
https://developers.google.com/web/tools/puppeteer/articles/ssr
```js
import puppeteer from 'puppeteer';

// In-memory cache of rendered pages. Note: this will be cleared whenever the
// server process stops. If you need true persistence, use something like
// Google Cloud Storage (https://firebase.google.com/docs/storage/web/start).
const RENDER_CACHE = new Map();

async function ssr(url) {
  if (RENDER_CACHE.has(url)) {
    return {html: RENDER_CACHE.get(url), ttRenderMs: 0};
  }

  const start = Date.now();

  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  try {
    // networkidle0 waits for the network to be idle (no requests for 500ms).
    // The page's JS has likely produced markup by this point, but wait longer
    // if your site lazy loads, etc.
    await page.goto(url, {waitUntil: 'networkidle0'});
    await page.waitForSelector('#posts'); // ensure #posts exists in the DOM.
  } catch (err) {
    console.error(err);
    throw new Error('page.goto/waitForSelector timed out.');
  }

  const html = await page.content(); // serialized HTML of page DOM.
  await browser.close();

  const ttRenderMs = Date.now() - start;
  console.info(`Headless rendered page in: ${ttRenderMs}ms`);

  RENDER_CACHE.set(url, html); // cache rendered page.

  return {html, ttRenderMs};
}

export {ssr as default};

```
#### SSR Example 1:
* https://developers.google.com/web/tools/puppeteer/articles/ssr#reuseinstance

```js
import express from `express`;
import {ssr} from `./ssr.mjs`;

app.get('/', async (req, res, next) => {
    const origin = `${req.protocol}://${req.get('host')}`;
    const html = await ssr(`${origin}/index.html`);
    return res.status(200).send(html).
});


```


#### For Debugging (in Head**ful** mode)
```js
const browser = await puppeteer.launch({
  headless: false // true | false :: headless | headful modes
});
```


#### Remote Debugging URL: `puppeteer.connect`
```js
 await puppeteer.connect({ __BROWSER_WEBSOCKET_ENDPOINT__ });
  // __BROWSER_WEBSOCKET_ENDPOINT__: Optional, Remote Debugging URL  .
  // Puppeteer's reconnects to the browser instance.
  // Otherwise, a new browser instance is launched.
```
#### Open a New Tab / New Page
```js
let page = await browser.newPage();
```

#### Configure Your View Port etc.
```js
await page.setViewport({
  width: 1338,
  height: 768,
  deviceScaleFactor: 3
});
```

#### Go to a site
```js
await page.goto(url, { waitUntil: 'domcontentloaded' }); //domcontentloaded | networkidle0
await page.goto('https://www.google.com');
```
#### Error Handling (`puppeteer.errors`)

returns: `<Object>`
`TimeoutError` <function> A class of TimeoutError.


Puppeteer methods might throw errors if they are unable to fulfill a request. For example, page.`waitForSelector(selector[, options])` might fail if the selector doesn't match any nodes during the given timeframe. For certain types of errors Puppeteer uses specific error classes. These classes are available via `puppeteer.errors`. An example of handling a timeout error:

```js
  try {
    await page.waitForSelector('.foo');
  } catch (e) {
    if (e instanceof puppeteer.errors.TimeoutError) {
      // Do something if this is a timeout.
    }
  }

```

#### Close Puppeteer
```js
await browser.close();
```

### Device Emulation
  Namespaces

    puppeteer.devices
    puppeteer.errors
    puppeteer.product

Returns a list of devices to be used with page.emulate(options). Actual list of devices can be found in src/common/DeviceDescriptors.ts.

```js
const puppeteer = require('puppeteer');
const iPhone = puppeteer.devices['iPhone 6'];

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.emulate(iPhone);
  await page.goto('https://www.google.com');
  // other actions...
  await browser.close();
})();
```


## Web Scraping With Puppeteer 
* Dont download all resources
* Get the DOM HTML

* https://hackernoon.com/tips-and-tricks-for-web-scraping-with-puppeteer-ed391a63d952
* https://developers.google.com/web/tools/puppeteer/articles/ssr#reuseinstance
