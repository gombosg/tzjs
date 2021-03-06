# tzjs
[![CircleCI](https://circleci.com/gh/dynasty-com/tzjs.svg?style=svg)](https://circleci.com/gh/dynasty-com/tzjs)
<!-- [![saucelabs](https://saucelabs.com/browser-matrix/tzjs.svg)](https://saucelabs.com/u/tzjs) -->

timezones without bloat

**use moment-timezone? save 400KB, minified!**



## quick start

```
npm i tzjs
```

```js
const d1 = new Date('2020-01-01') // Jan 1, midnight UTC
tzjs.getOffset('Europe/Berlin', d1) // -60

const d2 = new Date('2020-06-01') //  June 1, now we're on daylight savings
tzjs.getOffset('Europe/Berlin', d2) // -120

tzjs.getOffset('America/Los_Angeles', new Date()) // current offset in LA/SF
// Returns 420 or 480, depending on whether we're on daylight savings

const o1 = {hour: 'numeric', minute: '2-digit', timeZoneName: 'short'}
tzjs.fmt(o1).format(d1)
// example: "4:00 PM PST". localizes to the user's timezone and language

const o2 = {hour: 'numeric', minute: '2-digit', timeZone: 'UTC'}
tzjs.fmt(o2, 'en-US').format(d1)
// always returns "12:00 AM", regardless of browser settings

const o3 = {month: 'long', day: 'numeric', timeZone: 'UTC'}
tzjs.fmt(o3, 'es').format(d1)
// always returns "1 de enero"
```

support all the world's timezones, including DST and historical changes, in under 1KB of javascript!

`getOffset` is a few lines of very tricky code, working around a limitation in `window.Intl`.

`fmt` is a thin wrapper around `Intl.DateTimeFormat`. it's shorter to type and memoized, which we've measured to be important for performance.



## comparison

the two most popular libraries for dealing with dates in the browser are `moment` and `date-fns`.

if you needed to format times in a specific timezone--any timezone other than browser local or UTC--the situation was grim. until now!

| formatting library | min. size | timezone support lib | min. size |
|:-|:-|:-|:-|
| moment | 227KB | moment-timezone | 407KB |
| date-fns | 29KB | [doesn't exist yet](https://github.com/date-fns/date-fns/issues/489) | - |
| **window.Intl.DateTimeFormat** | **0KB** | **tzjs** | **1KB** |

**you can use `tzjs` by itself or together with `date-fns`**



## api reference


`tzjs` exports just three functions.


```js
import { getOffset, fmt, toDate } from 'tzjs'
```

or, with ES5,

```js
const tzjs = require('tzjs')
```

### `getOffset(timeZone, date)`

returns offset from UTC in minutes

note that if a timzone's at UTC+1, getOffset() returns -60, not 60

in other words, it returns the number of minutes you'd have to add to get to UTC

this behavior matches [Date.getTimezoneOffset](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/getTimezoneOffset)

**example:**

```js
getOffset('America/Los_Angeles', '2020-04-20')
// returns 420
```


### `fmt(options, locale)`

returns `Intl.DateTimeFormat(locale, options)`

memoizes the result. this is important, since the `DateTimeFormat` constructor is slow.

`locale` is optional and defaults to the browser locale.

**example:**

```js
const opts = {
  year: 'numeric',
  month: 'short',
  day: '2-digit',
  hour: 'numeric',
  minute: 'numeric'
}
const optsWithTz = Object.assign({timeZone: 'America/New_York'}, opts)

fmt(optsWithTz, 'en-US').format(new Date('2020-01-01T00:00:00Z'))
// always returns "Dec 31, 2019, 7:00 PM"

fmt(opts, 'en-US').format(new Date('2020-01-01T00:00:00Z'))
// returns "Dec 31, 2019, 4:00 PM" here in California
// return value depends on browser timezone

fmt(optsTz, 'es').format(new Date('2020-01-01T00:00:00Z'))
// always returns "31 dic. 2019 19:00"

fmt(opts).format(new Date('2020-01-01T00:00:00Z'))
// might produce any of the values above!
// return value depends on browser timezone and language setting
```


### `toDate`

helper function. takes a `Date` object, Unix millis, or an ISO timestamp like `"2020-01-01T00:00:00Z"`. returns a `Date`.

**example:**

```js
fmt({year: 'numeric'}).format(toDate('2020-06-01'))
// returns "2020"
```


## faq

### does it work in all browsers?

everywhere except Opera Mini and UC Browser for Android, mostly limited to China. https://caniuse.com/#search=datetimeformat

it even works in IE 11!

### why is `moment-timezone` so big?

it's bundling much of the [timezone database](https://en.wikipedia.org/wiki/Tz_database). this is big, complex and redundant: the browser already knows all of that information.

### so why not just use browser APIs?

the [Internationalization API](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl) has an omission so big that i think it qualifies as a bug.

you can **print** any instant in any timezone:

```js
const options = {
  hour: '2-digit',
  timeZoneName: 'short',
  timeZone: 'Europe/Berlin'
}
const format = window.Intl.DateTimeFormat('en-US', options)
format.format(new Date('2020-01-01')) // Midnight Jan 1 UTC
// "1 AM GMT+1"
format.format(new Date('2020-06-01')) // Midnight June 1 UTC
// "2 AM GMT+2" (because of DST)
```

so under the hood, browser must already have the timezone database, and must already be calculating offsets. however, the offsets are **not exposed in the API**.

### then how does tzjs work?

we work around this omission via a cute hack. we use `window.Intl` to format a given instant in both UTC and the target timezone. then, we parse both formatted times and subtract. turns out you can do this in just a few lines of code. see `index.js`.

### is it fast?

reasonbly. all that string parsing is slower than `moment` but fast enough that it would be 0% of your render time in most UIs.

on my 2011-era Thinkpad x220:

```
$ npm run bench
Computing offsets for 744 dates x 493 zones
tzjs.getOffset: 366792 offsets in 5946ms
momentTz.utcOffset: 366792 offsets in 386ms
```

that works out to 16us per offset for `tzjs` vs 1us for `moment-timezone`

`tzjs` saves you ~400KB from your minified bundle size, which can cut page load time significantly.



## date formatting

so the built-in `Intl.DateTimeFormat` is lots lighter than moment. however, writing `DateTimeFormat`s everywhere can get cumbersome.

not only that, it can be **slow**. we've seen the `DateTimeFormat` constructor show up in profiles, taking up more than half of our frontend CPU!


### solution: `tzjs.fmt`

we've found the following to work well. specifically, it's fast and light.

create a file that can be shared across your frontend. ours is called `date.js`:

```js
/* @flow */
import { fmt, toDate } from 'tzjs'

/**
 * Returns eg "Sun, Mar 11, 11:55pm PST"
 */
export function dateTimeTz (d: string | Date, timeZone?: ?string): string {
  return fmt('en-US', {
    weekday: 'short',
    month: 'short',
    day: 'numeric',
    hour: 'numeric',
    minute: 'numeric',
    hour12: true,
    timeZoneName: 'short',
    timeZone
  }).format(toDate(d))
}

/**
 * Returns eg "3/11" for March 11
 */
export function dateMD(d: string | Date, timeZone?: ?string): string {
  return fmt('en-US', {
    month: 'numeric',
    day: 'numeric',
    timeZone
  }).format(toDate(d))
}
```

then, elsewhere:


```js
/* @flow */
import { dateTimeTz } from '../date.js'

// ...

function MessageLabel (date: Date) {
  return <span>Sent {dateTimeTz(date, 'America/Los_Angeles')}</span>
}
```
