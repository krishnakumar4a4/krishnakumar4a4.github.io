+++
authors = [
    "Krishna Kumar Thokala",
]
title = "A Comparison between Rust and Erlang"
description = "A Comparison between Rust and Erlang"
date = 2018-03-13T09:43:12+05:30
tags = [
    "erlang",
    "rustlang",
    "infoq",
]
categories = [
    "erlang",
    "rustlang",
    "infoq",
]
series = ["erlang", "rustlang"]
aliases = ["erlang", "rustlang"]
images = []
+++
##### Key Takeaways

- Erlang provides lightweight processes, immutability, distribution with location transparency, message passing, supervision behaviors and many other high-level, dynamic features that make it great for fault-tolerant, highly available, and scalable systems.
- Unfortunately, Erlang is less than optimal at doing low-level stuff such as XML parsing, since dealing with anything that comes from outside of the Erlang VM into it is tedious
- For this kind of use cases, one could be tempted to consider a different language. In particular, Rust has recently come to the foreground due to its hybrid feature set, which makes similar promises to Erlang’s in many aspects, with the added benefit of low level performance and safety.
- While Rust and Erlang take completely different approaches on many key aspects of language design, including memory management, mutation, sharing, etc., where they deeply differ is at the Erlang BEAM level. The BEAM provides essential support for fault-tolerance, scalability and other foundational featurs of Erlang, which are not present in Rust.
- So, although Rust cannot be seen as a replacement for Erlang, it could make sense to mix both languages in the same project to leverage their strenghts.
  

> [Read the full comparison published on infoq](https://www.infoq.com/articles/rust-erlang-comparison/)  

> [Read the full comparison chinese translation on infoq](https://www.infoq.cn/article/rust-erlang-comparison)
