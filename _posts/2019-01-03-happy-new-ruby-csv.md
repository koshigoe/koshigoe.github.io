---
layout: post
title:  "Happy New Ruby CSV!"
date:   2019-01-03 09:00:00 +09:00
categories:
- Ruby
- CSV
---


The Ruby 2.6.0 released Dec 25. It includes a new CSV library([ruby/csv](https://github.com/ruby/csv) v3.0.2).

It said:

> - Improves CSV write performance. 3.0.2 will be about 2 times faster than 3.0.1.
> - Improves CSV parse performance for complex case. 3.0.2 will be about 2 times faster than 3.0.1.
>
> https://github.com/ruby/csv/blob/master/NEWS.md#302---2018-12-23

I'm so happy "2 times faster" if that is true. So, I confirmed this improvements.

It really faster, but not always "2 times faster". Even though, I'm so happy! Time is money!!

<script src="https://gist.github.com/koshigoe/b86d33c6dc3d55a5742d6feee88941f9.js"></script>

P.S. In my application, a parse logic got 2 times faster!!!
