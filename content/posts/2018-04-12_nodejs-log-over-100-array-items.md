+++
date = 2018-04-12T00:00:00.000Z
title = "Node.js: Log over 100 Array elements"
description = """
Use `util.inspect` with `maxArrayLength` set to `null` to display more than 100 items from an array.
"""
aliases = [
  "nodejs_log_over_100_aray_items"
]
[taxonomies]
topics = [ "Node.js" ]
[extra]
priority = 0.8
+++

Using `console.log` on an array with over 100 elements results in the following output

```js
// file.js
const arr = Array(500).fill('item');
console.log(arr)
```
```
node file.js

[ 'item',
  'item',
  'item',
  <...cut...>
  'item',
  'item',
  'item',
  ... 400 more items ]
```

Only first 100 are displayed with a message that there are more elements.

Use `util.inspect` with `maxArrayLength` set to `null` to display more than 100 items from an array:

```js
const util = require('util')
console.log(util.inspect(array, { maxArrayLength: null }))
```

By default, in Node.js `maxArrayLength` is set to `100`.