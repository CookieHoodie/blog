---
title: "How to Create a Logseq Plugin"
date: 2023-04-05T12:41:59+08:00
showToc: true
TocOpen: false
ShowReadingTime: true
cover:
    image: "images/cover.png" 
    alt: "Logseq plugin" 
    hidden: false 
---

## Introduction
Ever since I discovered [Logseq](https://logseq.com/), I have been using it as my daily note taking tool. It has a lot of powerful features, and if you think something is missing, you could easily find or write a plugin that suits your needs. In this post, I will show you how to write your own Logseq plugin.

We will be implementing a command that we could use in the Logseq editor. In particular, the command will fetch the current market price data of a stock from the internet and display it in the editor:
![Demo GIF](images/demo.gif)
Fun fact: I personally wrote and used this simple plugin to speed up my stock analysis workflow using Logseq📈. 

Let's get started!


## Prerequisites

Before we begin, make sure that you have the following tools installed:
- **Logseq Desktop App**: https://logseq.com/downloads
- **Node.js**: https://nodejs.org/en/download
    
   > If you don't want to download Node.js locally and clutter your device, check out how I use [Docker with VSCode]({{< ref "posts/tech/docker-vscode-development" >}}) to solve that!

## Getting Started
As of Logseq version 0.9.1, plugins are written using HTML, CSS, and JavaScript, similar to how you develop a website (or to be exact, a chrome extension). Here's roughly how it works, quoting from the official documentation:

> Currently, each plugin runs in an iframe separate from the main Logseq app. The main app provides some APIs (such as operations on the block, getting app settings) through the SDK to the plugins. The iframe serves as the plugin's UI and can be shown or hidden through related APIs. What's more, the main Logseq app provides ways to inject UI elements such as icons or buttons to some specific positions in the main app.

The command we will be implementing takes the form of `TICKER /GetPrice`. For example, running `AAPL /GetPrice` in Logseq will fetch the current market price of Apple stock and display it in the same block. Since it's only a simple command, we will only use JavaScript without going into HTML and CSS.

The final code could be found in [Resources](#resources) at the end of the page.

### Setting up Logseq
To be able to load our own plugin, we need to turn on developer mode in Logseq. To do so, click the three-dots menu, go to `Settings > Advanced` and turn on Developer Mode.

### Writing the Plugin
1. First, create a new folder where we will write our plugin in. Navigate into the folder. 

1. Create a file named `package.json` with the following content:
    ```json
    {
        "name": "Logseq Plugin Tutorial",
        "version": "0.1.0",
        "description": "Plugin Tutorial",
        "logseq": {
            "main": "dist/index.html"
        },
        "scripts": {
            "build": "parcel build --public-url . --no-source-maps index.html"
        }
    }
    ```
    The `name`, `version`, and `description` fields specify the metadata of our plugin. The `logseq.main` field specifies the entrypoint of the plugin, which we will generate later.

1. Install the dependencies by running:
    ```sh
    npm install --save-dev @logseq/libs parcel
    npm install axios cheerio
    ```
    `@logseq/libs` is mandatory for developing Logseq plugin, `parcel` is required to build our project to be loaded into Logseq, while the other 2 are only required for the purpose of our plugin.

1. Create a file named `index.html` with the following content:
    ```html
    <!DOCTYPE html>
    <html lang="en">
        <head>
            <meta charset="UTF-8" />
            <meta http-equiv="X-UA-Compatible" content="IE=edge" />
            <meta name="viewport" content="width=device-width, initial-scale=1.0" />
            <title>Logseq Plugin Tutorial</title>
        </head>
        <body>
            <script src="./index.js" type="module"></script>
        </body>
    </html>
    ```
    This file is where we could customize the UI of our plugin. But since we are not doing that, the important part here is the `<script>` tag, which loads and runs our code in `index.js` that we will create next.

1. Create a file named `index.js` with the following content:
    ```js
    import '@logseq/libs';
    import axios from 'axios';
    import * as cheerio from 'cheerio';

    // Retrieve content (ticker info) and block uuid from the current block.
    async function getTickerFromBlock() {
        const {content, uuid} = await logseq.Editor.getCurrentBlock();
        const ticker = content.trim();
        if (ticker === "") {
            throw new Error("Please specify the ticker name.");
        } else if (ticker.split(" ", 2).length > 1) {
            throw new Error(`Ticker ${ticker} is not valid.`);
        }
        return {ticker: ticker, blockUUID: uuid};
    }

    // webscrape roic.ai and get the stock price of the ticker
    async function getMarketPrice(ticker) {
        const response = await axios.request({method: "GET", url: `https://roic.ai/classic/${ticker}`});
        const $ = cheerio.load(response.data);
        // Hardcoded. Will stop working if the website layout changes!
        return $(`p:contains("RECENT ")p:contains("PRICE")`).next().text();
    }

    function main () {
        logseq.Editor.registerSlashCommand(
            "GetPrice",
            async () => {
                let tickerInfo = {};
                try {
                    tickerInfo = await getTickerFromBlock();
                } catch (err) {
                    await logseq.UI.showMsg(`${err.name} ${err.message}`, "error");
                    throw err;
                }

                try {
                    const price = await getMarketPrice(tickerInfo.ticker);
                    if (price === "") {
                        await logseq.UI.showMsg("Can't find the price info.", "error");
                    } else {
                        await logseq.Editor.updateBlock(tickerInfo.blockUUID, `Current market price: *${price}*`);
                    }
                }
                catch (err) {
                    await logseq.UI.showMsg(`${err.name} ${err.message}`, "error");
                    throw err;  // throw otherwise the message will pop up multiple times.
                }
            }
        )
    }

    logseq.ready(main).catch(console.error)
    ```
    In short, the above code: 

    - Import `@Logseq/libs` so that we could use the Logseq API.
    - Register function `main` to execute when Logseq is ready by calling `logseq.ready(main).catch(console.error)`.
    - Register the command `GetPrice`, which when called will run the subsequent code to get the stock market price.
    - Execute `getTickerFromBlock()` to get the content of the current block which the command is run. In our case, the content would be the stock ticker.
    - Execute `getMarketPrice(ticker)` which makes a HTTP request to webscrape and retrieve the market price for the given ticker. 

    Refer to the [Logseq API](https://plugins-doc.logseq.com/) to learn more about the different Logseq API calls used in the code.


1. Build the project by running:
    ```sh
    npm run build
    ```
    A `dist/index.html` will be generated. If you recall from step 1, this is the entrypoint of our plugin.

1. In the Logseq desktop application, click the three-dots menu, go to `Plugins > Load unpacked plugin` and select our project folder (i.e., where `package.json` is located).

1. Test out the plugin by writing `AAPL` in an empty block and run the command `/GetPrice` in the same block to see the current market price! 

### Some Notes
After writing the ticker name, we might need to wait for about 1 second before running the command, otherwise an error may be thrown. This is because Logseq will only save the content of a block shortly after we stop typing. This means if we run the command too fast without giving it time to save, the content will not be captured by the `logseq.Editor.getCurrentBlock()` function call and thus give an error.


## Summary
Logseq provides a powerful set of API that allows us to build useful plugins. There are a lot other things you could achieve with plugins through the same development process to customize your Logseq experience. By following the steps outlined in this blog, you can start creating your own plugins and exploring the full potential of Logseq.  

Give me a like👍if you find this useful! 

## Resources
- Code for this tutorial: https://github.com/CookieHoodie/tutorials/tree/main/logseq_plugin
- More plugin samples: https://github.com/logseq/logseq-plugin-samples
- Logseq API documentation: https://plugins-doc.logseq.com/
- Official plugin tutorial: https://docs.logseq.com/#/page/plugins%20101
