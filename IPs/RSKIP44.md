---
rskip: 44
title: Remove the zero-byte discount in data
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 1
created: 2017-06-24
---

# Remove the zero-byte discount in data

|RSKIP          |44           |
| :------------ |:-------------|
|**Title**      |Remove the zero-byte discount in data|
|**Created**    |24-JUN-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |1 |
|**Status**     |Draft | 

# **Abstract**

The current software does not compress transactions when transferred. Therefore currently the zero byte data discount allows blocks to be 17 times bigger, while consuming the same resources. This RSKIP removes this discount.

# Motivation

# Discussion

# Specification



[comment]: <> (The cost of any data byte is set to 68 gas units.)
