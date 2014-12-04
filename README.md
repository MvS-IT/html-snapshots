# [html-snapshots v0.6.3](http://github.com/localnerve/html-snapshots)
[![Build Status](https://api.travis-ci.org/localnerve/html-snapshots.png?branch=master)](http://travis-ci.org/localnerve/html-snapshots)
[![Coverage Status](https://img.shields.io/coveralls/localnerve/html-snapshots.svg)](https://coveralls.io/r/localnerve/html-snapshots?branch=master)
[![Dependency Status](https://david-dm.org/localnerve/html-snapshots.png)](https://david-dm.org/localnerve/html-snapshots)
[![devDependency Status](https://david-dm.org/localnerve/html-snapshots/dev-status.png)](https://david-dm.org/localnerve/html-snapshots#info=devDependencies)
[![Codacy Badge](https://www.codacy.com/project/badge/03d414fc2e264ef4b40456aae5b52108)](https://www.codacy.com/public/alex/html-snapshots)

> Takes html snapshots of your site's crawlable pages when an element you select is rendered.

## Overview
html-snapshots is a flexible html snapshot library that uses PhantomJS to take html snapshots of your webpages served from your site. A snapshot is only taken when a specified selector is detected visible in the output html. This tool is useful when your site is largely ajax content, or an SPA, and you want your dynamic content indexed by search engines.

html-snapshots gets urls to process from either a robots.txt or sitemap.xml. Alternatively, you can supply an array with completely arbitrary urls, or a line delimited textfile with arbitrary host-relative paths.

### Process Model
html-snapshots takes snapshots in parallel, each page getting its own PhantomJS process. Each PhantomJS process dies after snapshotting one page. You can limit the number of PhantomJS processes that can ever run at once with the `processLimit` option. This effectively sets up a process pool for PhantomJS instances. The default processLimit is 4 PhantomJS instances. When a PhantomJS process dies, and another snapshot needs to be taken, a new PhantomJS process is spawned to take the vacant slot. This continues until a `processLimit` number of processes are running at once.

## Breaking Change in v0.6.x
jQuery selectors are no longer supported by default. To restore the previous behavior, set the `useJQuery` option to `true`.
The upside is jQuery is no longer required to be loaded by the page being snapshotted. However, if you use jQuery selectors, or selectors not supported by [querySelector](https://developer.mozilla.org/en-US/docs/Web/API/document.querySelector), the page being snapshotted must load jQuery.

## More Information
Here are some [background and other notes](http://github.com/localnerve/html-snapshots/blob/master/docs/notes.md) regarding this project.

## Getting Started

### Installation
The simplest way to install html-snapshots is to use [npm](http://npmjs.org), just `npm
install html-snapshots` will download html-snapshots and all dependencies.

### Grunt Task
If you are interested in the grunt task that uses this library, check out [grunt-html-snapshots](http://github.com/localnerve/grunt-html-snapshots).

## Example Usage
Simple examples to demonstrate the options can be found in this section. A more in-depth usage example is located in this [article](http://github.com/localnerve/html-snapshots/blob/master/docs/example-heroku-redis.md) that includes explanation and code of a real usage featuring dynamic app routes, ExpressJS, Heroku, and more.

### Simple example
```javascript
var htmlSnapshots = require('html-snapshots');
var result = htmlSnapshots.run({
  source: "/path/to/robots.txt",
  hostname: "exampledomain.com",
  outputDir: "./snapshots",
  outputDirClean: true,
  selector: "#dynamic-content"
});
// result === true if snapshots were successfully started
```
This reads the urls from your robots.txt and produces snapshots in the ./snapshots directory. In this example, a selector named "#dynamic-content" appears in all pages across the site. Once this selector is visible in a page, the html snapshot is taken.

### Example - Per page selectors and timeouts
```javascript
var htmlSnapshots = require('html-snapshots');
var result = htmlSnapshots.run({
  input: "sitemap",
  source: "/path/to/sitemap.xml",
  outputDir: "./snapshots",
  outputDirClean: true,  
  selector: { 
    "http://mysite.com": "#home-content",
    "__default": "#dynamic-content"
  },
  timeout: { "http://mysite.com/superslowpage": 20000, "__default": 10000 }
});
// result === true if snapshots were successfully started
```
This reads the urls from your sitemap.xml and produces snapshots in the ./snapshots directory. In this example, a selector named "#dynamic-content" appears in all pages across the site except the home page, where "#home-content" appears \(the appearance of a selector in the output triggers the snapshot\). Finally, a default timeout of 10000 ms is set on all pages except http://mysite.com/superslowpage, where it waits 20000 ms.

### Example - Per page special output paths
```javascript
var htmlSnapshots = require('html-snapshots');
var result = htmlSnapshots.run({
  input: "sitemap",
  source: "/path/to/sitemap.xml",
  outputDir: "./snapshots",
  outputDirClean: true,
  outputPath: {
    "http://mysite.com/services/?page=1": "services/page/1",
    "http://mysite.com/services/?page=2": "services/page/2" 
  },
  selector: "#dynamic-content"
});
// result === true if snapshots were successfully started
```
This example implies there are a couple of pages with query strings in sitemap.xml, and we don't want html-snapshots to create directories with query string characters in the names. We would also have a rewrite rule that reflects this same mapping when `_escaped_fragment_` shows up in the querystring of a request so we serve the snapshot from the appropriate directory.

### Example - Per page selectors and jQuery
```javascript
var htmlSnapshots = require('html-snapshots');
var result = htmlSnapshots.run({
  source: "/path/to/robots.txt",
  hostname: "mysite.com",
  outputDir: "./snapshots",
  outputDirClean: true,
  selector: {
    "__default": "#dynamic-content",
    "/jqpage": "A-Selector-Not-Supported-By-querySelector"
  },
  useJQuery: {
    "/jqpage": true,
    "__default": false
  }
});
// result === true if snapshots were successfully started
```
This reads the urls from your robots.txt and produces snapshots in the ./snapshots directory. In this example, a selector named "#dynamic-content" appears in all pages across the site except in "/jqpage", where a selector not supported by [querySelector](https://developer.mozilla.org/en-US/docs/Web/API/document.querySelector) is used. Further, "/jqpage" loads jQuery itself \(required\). All the other pages don't need to use special selectors, so the default is set to `false`. Notice that since a robots.txt input is used, full URLs are **not** used to match selectors. Instead, paths \(and QueryStrings and any Hashes\) are used, just as specified in the robots.txt file itself.

### Example - Array
```javascript
var htmlSnapshots = require('html-snapshots');
var result = htmlSnapshots.run({
  input: "array",
  source: ["http://mysite.com", "http://mysite.com/contact", "http://mysite.com:82/special"],
  outputDir: "./snapshots",
  outputDirClean: true,  
  selector: "#dynamic-content"
});
// result === true if snapshots were successfully started
```
Generates snapshots for "/", "/contact", and "/special" from mysite.com. "/special" uses port 82. All use http protocol. Array input can be powerful, check out the complete [example](https://github.com/localnerve/html-snapshots/tree/master/examples/html5rocks).

### Example - Completion callback, Remote robots.txt
```javascript
var htmlSnapshots = require('html-snapshots');
var result = htmlSnapshots.run({
  source: "http://localhost/robots.txt",
  hostname: "localhost",
  outputDir: "./snapshots",
  outputDirClean: true,  
  selector: "#dynamic-content"
}, function(err, snapshotsCompleted) { 
  /* 
    Do something when html-snapshots has completed.

    err is undefined if all snapshots were generated successfully.

    snapshotsCompleted is an array of normalized paths to output files that 
      contain completed snapshots.
    If no snapshots are completed, this is an empty array.

    You can use snapshotsCompleted to populate shared storage if you are running
      in a scalable server environment with an ephemeral file system:
  */
  if (!err) {
    // safe to use snapshotsCompleted to update alternative storage
    //   Example: https://github.com/localnerve/wpspa/blob/master/server/workers/snapshots/lib/index.js
  }  
});
```
Generates snapshots in the ./snapshots directory for paths found in http://localhost/robots.txt. Uses those paths against "localhost" to get the actual html output. Expects "#dynamic-content" to appear in all output. The callback function is called when snapshots concludes.

### Example - Completion callback, Remote robots.txt, remove script tags from html snapshots
```javascript
var fs = require("fs");
var assert = require("assert");
var htmlSnapshots = require("html-snapshots");

var result = htmlSnapshots.run({
  source: "http://localhost/robots.txt",
  hostname: "localhost",
  outputDir: "./snapshots",
  outputDirClean: true,  
  selector: "#dynamic-content",
  snapshotScript: {
    script: "removeScripts"
  }
}, function(err, snapshotsCompleted) {
  var content;

  assert.ifError(err);

  snapshotsCompleted.forEach(function(snapshotFile) {
    content = fs.readFileSync(snapshotFile, { encoding: "utf8"});    
    assert.equal(false, /<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi.test(content));
    // there are no script tags in the html snapshots
  });
});
```
Same as previous example, but removes all script tags from the output of the html snapshot. Custom filters are also supported, see the customFilter Example in the explanation of the `snapshotScript` option. Also, check out the complete [example](https://github.com/localnerve/html-snapshots/tree/master/examples/custom).

## Options
Apart from the default settings, there are a number of options that can be specified. Options are specified in an object to the module's run method ``htmlSnapshots.run({ optionName: value })``.

+ `input` 
  + default: `"robots"`
  + Specifies the input generator to be used to produce the urls. 
      
      Possible *values*:
      
      `"sitemap"` Supply urls from a local or remote sitemap.xml file. Gzipped sitemaps are supported.
      `"robots"` Supply urls from a local or remote robots.txt file. Robots.txt files with wildcards are NOT supported - Use "sitemap" instead.      
      `"textfile"` Supply urls from a local line-oriented text file in the style of robots.txt      
      `"array"`, supply arbitrary urls from a javascript array.

+ `source`
  + default: `"./robots.txt"`, `"./sitemap.xml"`, `"./line.txt"`, or `[]`, depending on the input generator.
  + Specifies the input source. This must be a valid array or the location of a robots, text, or sitemap file for the corresponding input generator. robots.txt and sitemap.xml(.gz) can be local or remote. However, for the array input generator, this must be a javascript array of urls.

+ `sitemapPolicy`
  + default: `false`
  + For use only with the sitemap input generator. When true, lastmod and/or changefreq sitemap url child elements can be used to determine if a snapshot needs to be taken. Here are the possibilities for usage:
    + Both lastmod and changefreq tags are specifed alongside loc tags in the sitemap. In this case, both of these tags are used to determine if the url is out-of-date and needs a snapshot.
    + Only a lastmod tag is specified alongside loc tags in the sitemap. In this case, if an output file from a previous run is found for the url loc, then the file modification time is compared against the lastmod value to see if the url is out-of-date and needs a snapshot.
    + Only a changefreq tag is specified alongside loc tags in the sitemap. In this case, if an output file from a previous run is found for the url loc, then the last file modification time is used as a timespan \(from now\) and compared against the given changefreq to see if the url is out-of-date and needs a snapshot.
  
  Not all url elements in a sitemap have to have lastmod and/or changefreq \(those tags are optional, unlike loc\), but the urls you want to be able to skip \(if they are current\) must make use of those tags. You can intermix usage of these tags, as long as the requirements are met for making an age determination. If a determination on age cannot be made for any reason, the url is processed normally. For more info on sitemap tags and acceptable values, read the [wikipedia](http://en.wikipedia.org/wiki/Sitemaps) page.

+ `hostname`
  + default: `"localhost"`
  + Specifies the hostname to use for paths found in a robots.txt or textfile. Applies to all pages. This option is ignored if you are using the sitemap or array input generators.

+ `port`
  + default: 80
  + Specifies the port to use for all paths found in a robots.txt or textfile. This option is ignored if you are using the sitemap or array input generators.

+ `auth`
  + default: none
  + Specifies the old-school authentication portion of the url. Applies to all path found in a robots.txt or textfile.

+ `protocol`
  + default: `"http"`
  + Specifies the protocol to use for all paths found in a robots.txt or textfile. This option is ignored if you are using the sitemap or array input generators.

+ `outputDir`
  + default: none
  + **Required** \(you must specify a value\). Specifies the root output directory to put all the snapshot files in. Paths to the snapshot files in the output directory are defined by the paths in the urls themselves. The snapshot files are always named "index.html".

+ `outputDirClean`
  + default: `false`
  + Specifies if html-snapshots should clean the output directory before it creates the snapshots. If you are using sitemapPolicy and only specifying one of lastmod or changefreq in your sitemap \(thereby relying on file modification times on output files from a previous run\) this value must be false.

+ `outputPath`
  + default: none
  + Specifies per url overrides to the generated snapshot output path. The default output path for a snapshot file is rooted at outputDir, but is simply an echo of the input path - plus any arguments. Depending on your urls, your `_escaped_fragment_` rewrite rule (see below), or the characters allowed in directory names in your environment, it might be necessary to use this option to change the output paths.

      The value can be one of these *javascript types*:

      `"object"` If the value is an object, it must be a key/value pair object where the key must match the url (or path in the case of robots.txt style) found by the input generator.
      
      `"function"` If the value is a function, it is called for every page and passed a single argument that is the url (or path in the case of robots.txt style) found in the input.

  Notes: For pretty urls, this option may not be wanted/needed. The output file name is always "index.html".

+ `selector`
  + default: `"body"`
  + Specifies the selector to find in the output before taking the snapshot. The appearence of this selector in the output triggers a snapshot to be taken.
      
      The value can be one of these *javascript types*:
      
      `"string"` If the value is a string, it is used for every page.       
      
      `"object"` If the value is an object, it is interpreted as key/value pairs where the key must match the url (or path in the case of robots.txt style) found by the input generator. This allows you to specify selectors for individual pages. The reserved key "__default" allows you to specify the default selector so you don't have to specify a selector every individual page.
      
      `"function"` If the value is a function, it is called for every page and passed a single argument that is the url (or path in the case of robots.txt style) found in the input. The function must return a value to use for this option for the page it is given.
  
  NOTE: By default, selectors must conform to [this spec](http://www.w3.org/TR/selectors-api/#grammar), as they are used by [querySelector](https://developer.mozilla.org/en-US/docs/Web/API/document.querySelector). If you need selectors not supported by this, you must specify the `useJQuery` option, and load jQuery in your page.

+ `useJQuery`
  + default: `false`
  + Specifies to use jQuery selectors to detect when to snapshot a page. Please note that you cannot use these selectors if the page to be snapshotted does not load jQuery itself. To return to the behavior prior to v0.6.x, set this to `true`.
      
      The value can be one of these *javascript types*:
      
      `"boolean"` If the value is a boolean, it is used for every page. Note that if it is any scalar type such as "string" or "number", it will be interpreted as a boolean using javascript rules. Coerced string values "true", "yes", and "1" are specifically true, all others are false.
      
      `"object"` If the value is an object, it is interpreted as key/value pairs where the key must match the url (or path in the case of robots.txt style) found by the input generator. This allows you to specify the use of jQuery for individual pages. The reserved key "__default" allows you to specify a default jQuery usage so you don't have to specify usage for every individual page.
      
      `"function"` If the value is a function, it is called for every page and passed a single argument that is the url (or path in the case of robots.txt style) found in the input. The function must return a value to use for this option for the page it is given.
  
  NOTE: You do not *have to* use this option if your page uses jQuery. You only need this if your selector is not supported by [querySelector](https://developer.mozilla.org/en-US/docs/Web/API/document.querySelector). However, if you do use this option, the page being snapshotted must load jQuery itself.

+ `timeout`
  + default: 10000 (milliseconds)
  + Specifies the time to wait for the selector to become visible.
      
      The value can be one of these *javascript types*:
      
      `"number"` If the value is a number, it is used for every page in the website.      
      
      `"object"` If the value is an object, it is interpreted as key/value pairs where the key must match the url (or path in the case of robots.txt style) found by the input generator. This allows you to specify timeouts for individual pages. The reserved key "__default" allows you to specify the default timeout so you don't have to specify a timeout for every individual page.
      
      `"function"` If the value is a function, it is called for every page and passed a single argument that is the url (or path in the case of robots.txt style) found in the input. The function must return a value to use for this option for the page it is given.

+ `processLimit`
  + default: 4
  + Limits the number of child PhantomJS processes that can ever be actively running in parallel. A value of 1 effectively forces the snapshots to be taken in series (only one at a time). Useful if you need to limit the number of processes spawned by this library. Experiment with what works best. One guideline suggests about [4 per CPU](http://stackoverflow.com/questions/9961254/how-to-manage-a-pool-of-phantomjs-instances).

+ `checkInterval`
  + default: 250 (milliseconds)
  + Specifies the rate at which the PhantomJS script checks to see if the selector is visible yet. Applies to all pages.

+ `pollInterval`
  + default: 500 (milliseconds)
  + Specifies the rate at which html-snapshots checks to see if a PhantomJS script has completed. Applies to all pages.

+ `snapshotScript`
  + default: This library's [default](https://github.com/localnerve/html-snapshots/blob/master/lib/phantom/default.js) snapshot script. This script runs in PhantomJS and takes the snapshot when the supplied selector becomes visible.
  + Specifies the PhantomJS script to run to actually produce the snapshot. The script supplied in this option is run per url (or path) by html-snapshots in a separate PhantomJS process. Applies to all pages.

    The value can be one of these *javascript types*:

    `"string"` If the value is a string, it must an absolute path to a custom PhantomJS script you supply. html-snapshots will spawn a separate PhantomJS instance to run your snapshot script and give it the following [arguments](http://phantomjs.org/api/system/property/args.html):
      + `system.args[0]` The path to your PhantomJS script.
      + `system.args[1]` The full path to the output file where your script writes the html.
      + `system.args[2]` The url to snapshot.
      + `system.args[3]` The selector to watch for to signal page completion.
      + `system.args[4]` The overall timeout \(milliseconds\).
      + `system.args[5]` The interval \(milliseconds\) to watch for the selector.
      + `system.args[6]` A flag indicating jQuery selectors should be supported.

    `"object"` If an object is supplied, it has the following properties:
      + `script` This must be one of the following values:
        + `"removeScripts"` This runs the default snapshot script with an output filter that removes all script tags are removed from the html snapshot before it is saved.
        + `"customFilter"` This runs the default snapshot script, but allows you to supply any output filter.
      + `module` This property is required only if you supplied a value of `"customFilter"` for the `script` property. This must be an absolute path to a PhantomJS module you supply. Your module will be `require`d and called as a function to filter the html snapshot output. Your module's function will receive the entire raw html content as a single input string, and must return the filtered html content.

      customFilter Example:
      ```javascript
      // option snippet showing snapshotScript object with "customFilter":
      {
        snapshotScript: {
          script: "customFilter",
          module: "/path/to/myFilter.js"
        }
      }

      // in myFilter.js:
      module.exports = function(content) {
        return content.replace(/someregex/g, "somereplacement"); // remove or replace anything
      }
      ```

+ `phantomjs`
  + default: A package local reference to phantomjs.
  + Specifies the phantomjs executable to run. Override this if you want to supply a path to a different version of phantomjs. To reference PhantomJS globally in your environment, just use the value, "phantomjs". Remember, it must be found in your environment path to execute.
See [PhantomJS](http://phantomjs.org/) for more information.

## Example Rewrite Rule
Here is an example apache rewrite rule for rewriting \_escaped\_fragment\_ requests to the snapshots directory on your server.
```
<ifModule mod_rewrite.c>
  RewriteCond %{QUERY_STRING} ^_escaped_fragment_=(.*)$
  RewriteCond %{REQUEST_URI} !^/snapshots [NC]
  RewriteRule ^(.*)/?$ /snapshots/$1 [L]
</ifModule>
```
This serves the snapshot to any request for a url (perhaps found by a bot in your robots.txt or sitemap.xml) to the snapshot output directory. In this example, no translation is done, it simply takes the request as is and serves its corresponding snapshot. So a request for `http://mysite.com/?_escaped_fragment_=` serves the mysite.com homepage snapshot.

You can also refer `_escaped_fragment_` requests to your snapshots in ExpressJS with a similar method using [connect-modrewrite](https://github.com/tinganho/connect-modrewrite) middleware. Here is an analogous example of a connect-modrewrite rule:
```javascript
  '^(.*)\\?_escaped_fragment_=.*$ /snapshots/$1 [NC L]'
```

Another ExpressJS middleware example using html-snapshots can be found at [wpspa/server/middleware/snapshots.js](https://github.com/localnerve/wpspa/blob/master/server/middleware/snapshots.js)

## License
This software is free to use under the LocalNerve, LLC MIT license. See the [LICENSE file](https://github.com/localnerve/html-snapshots/blob/master/LICENSE) for license text and copyright information.

Third-party open source code used are listed in the [package.json file](https://github.com/localnerve/html-snapshots/blob/master/package.json).