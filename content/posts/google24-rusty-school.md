+++
title = 'Google24 - RUSTY SCHOOL'
summary = 'Rust Reversing Challenge @ Google CTF 2024'
date = 2024-07-28T21:48:12-04:00
tags = ['CTF', 'writeups', 'reverse']
draft = True
+++
> *REVERSE, 406 (6 solves)*

This was a **RUST** reverse challenge. Unfortunately, I wasn't able to solve the challenge during the ctf but was able to continue working on it and figure out what I missed during the competition.

## Problem Description
We were given an [attachment](https://storage.googleapis.com/2024-attachments/5da5785fde1d540697e26cf17163031f5e9e4e22b4e3612552c35947734372a34700fce9dade65b151a5bfcaceaa87b4b20b703703d089a0cd3416333833744b.zip), an binary and an ecrypted(?) file, and the following description:

> My school asked me to create a new encryption algorithm. ...But I haven't done any crypto for years. My skills are a little bit rusty... Nevertheless, I managed to write this cool encryption tool. It works (well that's at least what I'm claiming) and it seems to be impossible to decrypt the ciphertext :evil. Or it's not???
> 
> Note: The original plaintext is ascii printable so you can reduce the search space.

## Reversing
Based on the given files, it was obvious that the goal is to figure out the encryption routine and then decrypt [flag.txt.encrtyped](TODO). However, just like all rust binary, the decompile result from all existing tools were hard to understand...

### Try figuring stuff?
Thankfully, the binary was not stripped and I was able to start from symbols that caught my eye.

## Solution

### Routine
Based on the reversing, I was able to figure out 
![](https://sketchviz.com/@mhsuab/b6b5097c6328782298916df3479f5761/a391d03134d3e8c1a11ca8b10147494be6183079.png)

