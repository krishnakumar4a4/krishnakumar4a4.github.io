---
title: rust-java-rust
date: 2019-03-24
images: []
tags: ["rustlang","java"]
githubUrl: "https://github.com/krishnakumar4a4/rust-java-rust"
summary: "Provides sample code to call rust methods from embedded JVM without need for dynamic library load"
---
For Interoperability between rust and java

Special about this repo: Do not require rust library to be dynamically loaded for calling methods on rust.

### Features:
- JVM is started as embedded from the rust program. 
- Java methods can be called from rust.
- Rust methods are registered as native functions to JVM and which can then be called from Java. 

#### Uses rucaja, jni-rs and jni crates.
