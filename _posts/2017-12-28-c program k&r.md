---
layout: post
title:  "The C Programming K&R Notes"
date:   2017-12-28 17:10:00 +0800
categories: 学习
tags:   C
description: K&R Notes
---
记录一下K&R里的点，最近写C程序，再读一遍K&R。。。
* An enumeration is a list of constant integers
* For example, suppose that int is 16 bits and long is 32 bits.Then -1L < 1U, because 1U, which is an unsigned int, is promoted to a signed long.But -1L > 1UL, because -1L is promoted to an unsigned long and thus appears to be a large postive number.
* ++n increment n before its value is used, while n++ increments n after its value has been used.

 {% highlight c %}
	void swap(int x, int y) {
		int temp;
		temp = x;
		x = y;
		y = temp;
	}
{% endhighlight %}

Call by value, swap can't affect the arguments *a* and *b* in the routine that called it.The function above `swaps copies of a and b`.
