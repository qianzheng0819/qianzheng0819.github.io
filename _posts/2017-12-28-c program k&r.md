---
layout: post
title:  "The C Programming K&R Notes"
date:   2017-12-28 17:10:00 +0800
categories: 学习
tags:   C
description: K&R Notes
---
## 记录一下K&R里的点，最近写C程序，再读一遍K&R。。。
* An enumeration is a list of constant integers
* For example, suppose that int is 16 bits and long is 32 bits.Then -1L < 1U, because 1U, which is an unsigned int, is promoted to a signed long.But -1L > 1UL, because -1L is promoted to an unsigned long and thus appears to be a large postive number.
