# Use puppeteer to download a file:

## Using an `iframe`
**Description:**


**Credits:**
https://github.com/puppeteer/puppeteer/issues/1888#issue-291077489
https://github.com/puppeteer/puppeteer/issues/299#issuecomment-322862535


- This is a script that
  - Gets a page
  - Clicks a button on it
  - When the button is clicked, the page inserts an invisible iframe with a source to a download file.
  - Chrome downloads the file.
  - Now, you need to look for the request going off and then save the buffer of that response body.
    - Documentation: https://github.com/puppeteer/puppeteer/blob/main/docs/api.md#httpresponsebuffer



```js
const puppeteer = require('puppeteer');
  
async function main() {
    const browser = await puppeteer.launch({headless: false, args: ['--no-sandbox']});
    const page = await browser.newPage();
    await page.setViewport({width:1920, height: 1080});

    await page.setRequestInterception(true);
    page.on('request', req => { console.log('Request:', req.url()); req.continue() });
    page.on('response', resp => console.log('Response:', resp.url()));

    await page.goto('https://damjan.softver.org.mk/pupp/', {waitUntil: 'networkidle2'});
    console.log('wait a bit and click…');
    await page.waitFor(1000);
    await page.click('button');

    console.log('wait a bit and say goodbye…');
    await page.waitFor(2*1000);
    await browser.close();
}

(async () => { await main() })()
```



## Catch the responses and write the files to a location of your choice
**Credits:** https://github.com/puppeteer/puppeteer/issues/299#issuecomment-328295644

**Pro tip:** You may need to adjust the timing for your page. Waiting for the load event and `networkidle` might not be enough.

```js
const puppeteer = require('puppeteer');
const fs = require('fs');
const mime = require('mime');
const URL = require('url').URL;

(async() => {
const browser = await puppeteer.launch();
const page = await browser.newPage();

const responses = [];
page.on('response', resp => {
  responses.push(resp);
});

page.on('load', () => {
  responses.map(async (resp, i) => {
    const request = await resp.request();
    const url = new URL(request.url);

    const split = url.pathname.split('/');
    let filename = split[split.length - 1];
    if (!filename.includes('.')) {
      filename += '.html';
    }

    const buffer = await resp.buffer();
    fs.writeFileSync(filename, buffer);
  });
});

await page.goto('https://news.ycombinator.com/', {waitUntil: 'networkidle'});
browser.close();
})();
```


## Download a file if the URL is known before-hand

**Credits:** https://github.com/puppeteer/puppeteer/issues/299#issuecomment-338240864

* Call fetch in an `evaluate(...)`
* `downloadUrl` is a string with the URL of the file you want to download.
* You can then use `downloadedContent` to write to a file.

```js
const downloadedContent = await page.evaluate(async downloadUrl => {
  const fetchResp = await fetch(downloadCsvUrl, {credentials: 'include'});
  return await fetchResp.text();
}, downloadUrl);
```

## Response Interception
* https://www.youtube.com/watch?v=EufdahOSJIc&feature=youtu.be&t=31m18s
* by https://github.com/umaar

### Use `request` event and `response` method
**Credits:** https://github.com/puppeteer/puppeteer/issues/599#issuecomment-357458054
```js
page.on('request', (request) => {
    if (request.url() === "http://abc.com/a.js") {
        request.respond({
            status: 200,
            contentType: 'application/javascript; charset=utf-8',
            body: 'console.log(1);'
        });
    } else {
        request.continue();
    }
});
```
### Another work-around
**Credits:** https://github.com/puppeteer/puppeteer/issues/1229#issuecomment-357469434

```js
  import fetch from 'node-fetch'

  const requestInterceptor = async (request) => {
    const url = request.url()
    const requestHeaders = request.headers()
    const acceptHeader = requestHeaders.accept || ''
    if (url.includes('some-website-with-csp.com') && (acceptHeader.includes('text/html'))) {
      const cookiesList = await page.cookies(url)
      const cookies = cookiesList.map(cookie => `${cookie.name}=${cookie.value}`).join('; ')
      delete requestHeaders['x-devtools-emulate-network-conditions-client-id']
      if (requestHeaders.Cookie) {
        requestHeaders.cookie = requestHeaders.Cookie
        delete requestHeaders.Cookie
      }
      const theseHeaders = Object.assign({'cookie': cookies}, requestHeaders, {'accept-language': 'en-US,en'})

      const init = {
        body: request.postData(),
        headers: theseHeaders,
        method: request.method(),
        follow: 20,
      }
      const result = await fetch(
        url,
        init,
      )
      const resultHeaders = {}
      result.headers.forEach((value, name) => {
        if (name.toLowerCase() !== 'content-security-policy') {
          resultHeaders[name] = value
        } else {
          console.log('CSP', `omitting CSP`, {originalCSP: value})
        }
      })
      const buffer = await result.buffer()
      await request.respond({
        body: buffer,
        resultHeaders,
        status: result.status,
      })
    } else {
      request.continue();
    }
  }

  await page.setRequestInterception(true)
  page.on('request', requestInterceptor)
```




