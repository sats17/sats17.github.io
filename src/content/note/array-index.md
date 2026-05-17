---
title: Why Array Index Access is O(1)
description: Information about the bounding box of an object.
publishDate: "2022-09-14T04:23:00Z"
---

We know that arrays are stored inside RAM as contiguous memory blocks.
So, when we store an array of size 13, let’s say it starts at memory address 2001. If we want to access index 7,
we already know that index 0 is located at address 2001, so we can simply calculate the address by adding 2001 + 7 to find where index 7 is stored.

This is how arrays achieve O(1) time complexity for index-based access using simple mathematical calculations.
