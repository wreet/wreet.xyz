---
layout: post
title:  "newt 0.6.0 now available"
date:   2019-03-29 11:05:25
categories: newt fuzzing
---

I'm happy to announce that the latest version of newt, 0.6.0, is [now available](https://github.com/wreet/newt)! This new version introduces many bug and stability fixes, as well as three new mutator modes to use when fuzzing files. Below I'll list some of the changes:

* Added the ability to select specific mutators when format fuzzing
* Added the ability to fuzz input from stdin
* Added the ability to generate fuzzy buffers to use with other fuzzing mechanisms
* Added three new mutators, byte arithmetic, bit rotate and "ripple" mutation
* Added logging for which mutators are being used
* Fixed error where some mutators discarded user-specified fuzz factor
* Fixed issue in "buffmangler" mutator that sometimes generated bad cases
* Fixed issue where certain program exit codes confused newt process monitor
* Fixed issue with procmon that caused gdb-monitored programs to not respawn
* Improved help messages to be more useful
* Updated readme with a few usage examples

## "ripple" mutation

Something I am very excited about in this release is a new mutator I am calling "ripple" mutation. It is based on the byte arithmetic methodology, wherein a byte value is selected and changed by random amounts. The difference is that after selecting an "impact" byte, arithmetic is also performed around that byte in both directions decreasing by squares from the original change. I like to think of this as like what happens when you throw a stone into a pond, and this is where the mutator's name comes from.

In my initial testing, this new mutation mode has proven itself to be the most effective in newt's arsenal on a variety of formats including  fonts, images, videos and especially PDFs. I'm very pleased with the results so far, and I hope you will find it useful as well. 

For a simple walkthrough on how to use newt, check out [this earlier post](https://wreet.xyz/2019/03/04/simple-fuzzing-with-newt/). 
