---
date: 2019-7-21
tag:
  - Frontend
  - JavaScript
---

# Json-server with multiple files

First of all, I'm not a native speaker of English.

## Intro

First, let's take a look at [json-server#getting-started](https://github.com/typicode/json-server#getting-started) :

Create a `db.json` file with some data

```
{
  "posts": [
    { "id": 1, "title": "json-server", "author": "typicode" }
  ],
  "comments": [
    { "id": 1, "body": "some comment", "postId": 1 }
  ],
  "profile": { "name": "typicode" }
}
```

Start JSON Server

```
json-server --watch db.json
```

Now if you go to http://localhost:3000/posts/1, you'll get

```
{ "id": 1, "title": "json-server", "author": "typicode" }
```

It's pretty simple. However, the data might be more complicated in real world, so I'm going to create a file for each api.

## Starting

當下版本為 `"json-server": "^0.15.0"`

### Structure with multiple files

Create a new folder `mock`, and add a script in `package.json`:

```json
"scripts": {
  "mock": "json-server --watch ./mock/db.js"
}
```

First, transform `db.json` into `db.js`：

```JavaScript
// db.js
// Should export a function which return an object
module.exports = () => ({
  posts: [{ id: 1, title: "json-server", author: "typicode" }],
  comments: [{ id: 1, body: "some comment", postId: 1 }],
  profile: { name: "typicode" }
});
```

It means I can write some stuff like：

```JavaScript
// db.js
const posts = require("./posts");
const comments = require("./comments");
const profile = require("./profile");
module.exports = () => ({
  posts,
  comments,
  profile
});
```

Create corresponding files, for example：

```JavaScript
// posts.js
module.exports = [{ id: 1, title: "json-server", author: "typicode" }];
```

### Import files automatically

It's annoyed that I have to import the file whenever I create one.

Use [glob](https://github.com/isaacs/node-glob) to get file names：

```JavaScript
// db.js
const Path = require("path");
const glob = require("glob");
const apiFiles = glob.sync(Path.resolve(__dirname, "./") + "/**/*.js", {
  nodir: true
});
// apiFiles will be :
// [ '/Users/billy/Desktop/json-server-multiple-files-sample/mock/comments.js',
//   '/Users/billy/Desktop/json-server-multiple-files-sample/mock/db.js',
//   '/Users/billy/Desktop/json-server-multiple-files-sample/mock/posts.js',
//   '/Users/billy/Desktop/json-server-multiple-files-sample/mock/profile.js' ]

let data = {};

apiFiles.forEach(filePath => {
  const api = require(filePath);
  let [, url] = filePath.split("mock/"); // e.g. comments.js
  url = url.slice(0, url.length - 3)  // remove .js >> comments
  data[url] = api;
});

// data will be :
// { 'comments': [ { id: 1, body: 'some comment', postId: 1 } ],
//   'db': {},
//   'posts': [ { id: 1, title: 'json-server', author: 'typicode' } ],
//   'profile': { name: 'typicode' } }

module.exports = () => {
  return data;
};
```

But I don't need `db.js` to be a API.<br>
Rename `db.js > _db.js` and add a rule to prevent all files with `_` prefix from transforming API routes:

```JavaScript
// db.js
// ...
const apiFiles = glob.sync(Path.resolve(__dirname, "./") + "/**/[!_]*.js", {
// ...
```

### Monitor multiple files

Since json-server will automatically restart only when `_db.js` is changed, I have to do it whenever adding a API or modify data.
[Solution](https://github.com/typicode/json-server/issues/434#issuecomment-273498639)：

```
yarn add -D nodemon
```

```json
"scripts": {
  "mock": "nodemon --watch mock --exec 'json-server ./mock/_db.js'"
}
```

### Better router

What if APIs become more complicated:

```
/blog/posts
/blog/comments
/blog/profile
/documents/query
```

The mock folder structure:

```
├── _db.js
├── blog
│   ├── comments.js
│   ├── posts.js
│   └── profile.js
└── documents
    └── query.js
```

Modify rules of `_db.js`:

```JavaScript
// db.js
const Path = require('path');
const glob = require('glob');
const config = require('./config.json');
const apiFiles = glob.sync(
  Path.resolve(__dirname, './') + '/**/[!_]*.js',
  {
    nodir: true,
  },
);

let data = {};


apiFiles.forEach(filePath => {
  const api = require(filePath);
  let [, url] = filePath.split('mock/');
  url = url.slice(0, url.length - 3);
  data[url.replace(/\//g, '-')] = api; // 只有這裡更動
});

// data會是
// { 'blog-comments': [ { id: 1, body: 'some comment', postId: 1 } ],
//   'blog-posts': [ { id: 1, title: 'json-server', author: 'tydpicode' } ],
//   'blog-profile': { name: 'typicode' },
//   'documents-query': { data: 123 } }

module.exports = () => {
  return data;
};
```

create `config.json` `router.json` files：
（[config usage](https://github.com/typicode/json-server#cli-usage)[router usage](https://github.com/typicode/json-server#add-custom-routes)）

```json
// config.json
{
  "port": 9898,
  "routes": "./mock/router.json"
}
```

```json
// router.json
{
  "/*/*/*": "/$1-$2-$3",
  "/*/*": "/$1-$2"
}
```

Now `/documents/query` will correspond to `/documents-query`.<br>
If `/documents/query/something` is existed, it'll correspond to `/documents-query-something`.That's totally handle the folder structure.

Last modified：

```JavaScript
// _db.js
const Path = require("path");
const glob = require("glob");
const apiFiles = glob.sync(Path.resolve(__dirname, "./") + "/**/[!_]*.js", {
  nodir: true
});

let data = {};
apiFiles.forEach(filePath => {
  const api = require(filePath);
  let [, url] = filePath.split("mock/");
  url =
    url.slice(url.length - 9) === "/index.js"
      ? url.slice(0, url.length - 9) // remove /index.js
      : url.slice(0, url.length - 3); // remove .js
  data[url.replace(/\//g, "-")] = api;
});
module.exports = () => {
  return data;
};
```

If there's a API route `/documents`, I can restructure

```
├── _db.js
├── blog
│   ├── comments.js
│   ├── posts.js
│   └── profile.js
├── config.json
├── documents
│   └── query.js
├── documents.js
└── router.json
```

to be something like：

```
├── _db.js
├── blog
│   ├── comments.js
│   ├── posts.js
│   └── profile.js
├── config.json
├── documents
│   ├── index.js
│   └── query.js
└── router.json
```

## Others

### Decode Chinese characters

If there's a API route `documents/基本`, I won't be able to visit it.
`基本`will be encode to be`%E5%9F%BA%E6%9C%AC`. Since we know the encoded string is consisted of the percent character "%" followed by two hexadecimal digits, I can create a middleware to handle it. Use prefix `_` which I've reserved. Decode url when the url contains the encoded string:

```
├── _db.js
├── blog
│   ├── comments.js
│   ├── posts.js
│   └── profile.js
├── config.json
├── documents
│   ├── index.js
│   ├── query.js
│   └── 基本.js
├── middlewares
│   ├── _decodeChinese.js
└── router.json
```

```JavaScript
// _decodeChinese.js
module.exports = function(req, res, next) {
  const regex = /%(\d|[A-Z]){2}/;
  const isMatch = regex.test(req.url);
  if (isMatch) {
    req.url = decodeURI(req.url);
  }
  next();
};
```

```json
// config.json
{
  // ...
  "middlewares": ["./mock/middlewares/_decodeChinese.js"]
  // ...
}
```

### Get data using POST

Sometimes People have to get data by POST instead of GET, but `custom route` of `json-server` will always be GET.
[Solution](https://github.com/typicode/json-server/issues/453#issuecomment-343048811)<br>
Again `middleware`. To detect whenever the method is POST, transform it into GET.

```JavaScript
// _postAsGet.js
module.exports = function(req, res, next) {
  if (req.method === 'POST') {
    // Converts POST to GET and move payload to query params
    // This way it will make JSON Server that it's GET request
    req.method = 'GET';
    req.query = req.body;
  }
  // Continue to JSON Server router
  next();
};
```

```json
// config.json
{
  // ...
  "middlewares": ["./mock/middlewares/_postAsGet.js"]
  // ...
}
```

[You can find a template here.](https://github.com/newsbielt703/json-server-mulitple-files-sample)