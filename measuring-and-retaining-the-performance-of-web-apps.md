# Measuring and retaining the performance of web-apps

How do you know whether your application performs well? Performance can be many things, it can be how fast certain methods in your software are executed, or it can be whether or not your users experience that buttery smooth 60 FPS when they scroll. In this article, I will focus on UI performance, but the tool I’m introducing could just as well be used to measure the execution time of functions. 
But how do we measure performance, and even better, how can we ensure that the performance doesn’t go haywire as a result of a developer unwillingly introducing logic that results in “layout trashing” or increased time complexity.
Focusing on UI performance, one could pop up the Google Chrome devtools and record usage of the web-app to ensure that it doesn’t do unnecessary calculations (CSS calculations or JavaScript), or we could use the FPS-meter to ensure that the FPS doesn’t drop significantly upon scrolling. While these things are pretty easy to do, it isn’t trivial to perform these checks programmatically in a CI/CD environment. A few different tools exist for doing it programmatically, and I even created my own library for it called tracealyzer [https://github.com/madskonradsen/tracealyzer](https://github.com/madskonradsen/tracealyzer), but since more comprehensive tools exist, what I will recommend instead is the library Tracelib. Tracelib is a small NPM package that can analyze trace-files from Chrome devtools. Let's get started by installing tracelib and puppeteer. Puppeteer will be used to automated Chrome, and fetch the trace-file programmatically.

```
npm i -g tracelib puppeteer
```

Tracelib is a comprehensive library that provides several useful functions such as `getSummary` that outputs how long different parts of the page-load process took, such as rendering, painting, scripting, and such. Then we have `getWarningCounts` which for example returns the number of "ForcedLayouts" which is what we use to determine whether the page exhibits "layout trashing". Next is `getMemoryCounters` which we can use to fetch information about the Heap-size, JS Event Listeners, and more, which is particularly useful for detecting leaks. Then we have the `getDetailStats` function which returns timestamp and values of the different events such as scripting, painting, and rendering. Finally, we have `getFPS` which is what I'm going to use throughout the rest of this article.

First, we import the two libraries and define a path for our trace-file. Then we launch a page in puppeteer and start the tracing after the load is done (this is to only measure FPS when the user is scrolling). Then we use window.scrollTo to smoothly scroll to the bottom of the page (please note that window.scrollTo doesn't return a Promise, and thus can't have a leading "await", this is only for demonstration purpose). After that, we can stop the tracing and close the browser.
After we have generated our trace-file, we can pass it to Tracelib and invoke the `getFPS` function.

```javascript
import puppeteer from 'puppeteer';
import Tracelib from 'tracelib';

const TRACE_FILE = './tracefile.json';

(async () => {
    const browser = await puppeteer.launch();
    const page = await browser.newPage();
    await page.goto('https://www.wired.com');
    await page.tracing.start({path: TRACE_FILE});

    await page.evaluate(() => {
        await window.scrollTo({ left: 0, top: document.body.scrollHeight, behavior: "smooth"});
    });

    await page.tracing.stop();
    await browser.close();

    const tasks = new Tracelib(TRACE_FILE);
    const fps = tasks.getFPS();
    console.log(fps);
})();
```

This will yield the following result:

```javascript
/**
 * {
 * times: [
 *  299959949.734,
 *  299959955.22,
 *  299960052.234,
 *  299960142.388,
 *  299960233.175,
 *  299960324.01,
 *  299960416.646,
 *  299960508.145,
 *  299960600.602,
 *  299960695.329
 * ],
 * values: [
 *  182.2821727559685,
 *  10.307790628308753,
 *  11.092131244032895,
 *  11.014792866762287,
 *  11.00897231503525,
 *  10.794939328106791,
 *  10.929081197725838,
 *  10.815838712204958,
 *  10.556652274643293,
 *  10.47987340271033
 * ]
 * }
 */
```

As seen above, we have generated a few measurements (with timestamps) which we can now analyze. We could for example calculate a mean FPS or find the median (puppeteer is often quite busy when tracing is just started, which often results in the outlier we see at index 0 (182.28...). It could also make sense to look at the standard deviation.
After the analysis, we now have some more meaningful numbers. But what should we do with them? One approach could be to simply use a Graphite set up and send the numbers to that. That way we could plot the numbers, and track how our FPS changes over time. Another approach could be to implement simple "quality gates" in our CI -environment that for example made sure to fail builds that had an FPS below 30. That way, code that makes our FPS substantially worse would never even make it to our master-branch. An ideal setup would probably be to use a combination of the two. Continuously plotting a graph of the FPS, while a simply quality gate makes sure that the FPS doesn't get abnormally low.