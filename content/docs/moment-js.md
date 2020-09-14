---
title:  "moment.js"
date:   2017-01-23 16:30:07 +0200
categories: frontend javascript
---

# Introduction to moment.js

**Jan 23, 2017**<!-- \ -->
<!-- <sup>Last modified: **Dec 2, 2018**</sup> -->

Moment.js is a date library for Javascript. It has great features and is easy to use especially for those who are used to Python datetime objects. You can download it from <a href="http://momentjs.com/" target="_blank">momentjs.com</a> or directly import cdnjs link:

<a href="https://cdnjs.cloudflare.com/ajax/libs/moment.js/2.13.0/moment.min.js" target="_blank">https://cdnjs.cloudflare.com/ajax/libs/moment.js/2.13.0/moment.min.js</a>

Although it has many attributes, I believe its <a href="http://momentjs.com/docs/" target="_blank">documentation</a> is a bit complicated. I will try to explain the features that were useful for me recently.

# Basic Commands

First, let's get started by defining a moment object, <span style="text-decoration: underline;">now</span>. It gets current date and time.

```javascript
var now = moment();
```

You can also define a moment object with Unix <span style="text-decoration: underline;">timestamp</span>, either in milliseconds:

```javascript
var date = moment(1318781876721) // in milliseconds
```

or in seconds:

```javascript
var date = moment.unix(1318781876) // in seconds
var date = moment.unix(1318781876.721) // specify milliseconds in seconds
```

You can extract the <span style="text-decoration: underline;">number of day</span> in week, starting from Sunday.

```javascript
moment().day() // returns integer from 0 to 6, with Sunday equals 0
```

Also you can get the <span style="text-decoration: underline;">number of weekday</span> depending on your locale, with either first day of the week is Sunday or Monday. The first day has the value 0 and the last one gets 6.

```javascript
moment().weekday()
```

To set a specific day around today, use an argument in .day() as follows:

```javascript
moment().day(1) // represents tomorrow with the exact time of now
moment().day(-7) // represents this day a week ago
```

Or you can set a specific hour from a moment object:

```javascript
moment().hour(9).minute(30).second(0) // returns the moment of today at 09:30AM
```

To <span style="text-decoration: underline;">add</span> or <span style="text-decoration: underline;">subtract</span> to/from a date:

```javascript
var now = moment();
now.add(1, 'days'); // returns tomorrow with the exact time of now
now.subtract(1, 'weeks'); // returns last week with the exact time of now
```

# Comparing Dates & Times

When I used moment.js, I needed to compare two or more moments. For this kind of purposes, you can use these.

<span style="text-decoration: underline;">Difference</span>:

```javascript
var a = moment();
var b = moment().day(1);

b.diff(a); // returns 86400000, the number of milliseconds in 24 hours
b.diff(a, 'days'); // returns 1
```

<span style="text-decoration: underline;">Time to a date</span>, in string format:

```javascript
a.to(b); // returns "in a day"
```

<span style="text-decoration: underline;">Durations:</span>

```javascript
delta = moment.duration( b.diff(a) ); // returns a duration of 24 hours length
```

and for a duration of 1 day, 3 hours and 5 minutes, you can display it as follows:

```javascript
delta.days(); // returns 1
delta.hours(); // returns 3
delta.minutes(); // returns 5
```

By using all these features, I managed to calculate the time when the stock market opens or closes, and the time left until it opens or closes for a given arbitrary time. These are only a small fraction of what moment.js is capable of, if you feel like you need more of it, please visit its documentation page. I hope these will be useful for you as well, enjoy it!


<script src="https://utteranc.es/client.js"
        repo="developweekly/blog"
        issue-term="title"
        label="comments"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>