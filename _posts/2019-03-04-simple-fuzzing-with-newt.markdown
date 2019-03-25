---
layout: post
title:  "Setting up a simple fuzz run with newt"
date:   2019-03-04 12:20:25
categories: fuzzing newt tutorials
---

Fuzzing is a great way to find security and stability issues in software. At its most basic, it's extremely easy to do and generally requires much less work than auditing source code. Of course gathering a corpus and triaging bugs takes some time, but during the run itself you're free to do other things, which for me is the biggest advantage to this approach. 

This post will serve as a short tutorial for using my personal fuzzer, newt. It's a simple, unattended file format fuzzer written in Node, which features several mutators, automatic triaging, and can monitor processes using either gdb or Address Sanitizer. At the time of writing, newt is available publicly at version 0.5.4, however for the last several months I have been working on 0.6.0 which introduces many new features, including new mutators, and piles of bug fixes. I will make this new version available soon, however usage will remain essentially the same so this tutorial will be applicable to 0.6.0 as well. Newt is fully compatible with Linux and Mac OS, and with a little bit of work runs reasonably well on Windows. 

## Why use newt?

There are tons of great fuzzers out there, most with many more features than newt, so why use it at all? The best answer to this question is ease-of-use. Fuzzers like afl offer things that newt doesn't such as live run statistics, code coverage monitoring and crash minimizers(though this in particular is under active development). However many require a complex setup, such as recompiling with Address Sanitizer, and programs with GUIs may be especially challenging to set up as you'll typically need to edit the source code to make the fuzzer happy. With newt this is not necessary, though it also supports fuzzing like this if you wish. 

In writing newt, I wanted a tool that I could use without having to do much in the way of preparation. The idea was that if a simple fuzz run with newt uncovered a few bugs, then I knew it was worth my time to instrument a program or begin auditing source code in order to investigate further. I also wanted to come up with novel mutators in the hopes that it might uncover issues that other fuzzers had missed. 

## Installing newt

The latest version of newt will always be available on [my GitHub page](https://github.com/wreet/newt). First, we will clone the repository and install the required npm modules. If you don't already have Node on your machine, you'll need to install it now. I recommend using [nvm](https://github.com/creationix/nvm). 

{% highlight bash %}
$ git clone https://github.com/wreet/newt
$ cd newt
$ npm install
$ ./newt.js
#=> [~] newt.js 0.5.4 - a simple node-powered fuzzer
#=> Usage: newt command [opts]
#=> Commands:
#=>   autofuzz    Automatically generate cases and fuzz the subject
#=>   |  -i       Required, directory where file format or ngen seeds can be found
#=>   |  -o       Required, output directory for crashes, logs, cases
#=>   |  -s       Required, the subject binary to fuzz
#=>   |  -k       Sometimes required, kill subject after -k seconds, useful for GUI bins
#=>   |  -f       Optional, int value that translates to fuzzing 1/-f byte in buffMangler mode  
#=>   |  -m       Optional, monitor mode. Default is gdb, asan instrumented bins also supported                                                                                     
#=>   procmon     Launch and monitor a process
#=>   |  -s       Required, the subject binary [with args]
#=>   |  -m       Required, monitor mode [asan|gdb]
#=>   |  -r       Optional, respawn process on exit
#=>   |  -o       Optional, output dir. Results printed to console if none specified
#=>   netfuzz     Fuzz a remote network service
#=>   |  -i       Required, directory where ngen seeds can be found
#=>   |  -o       Required, output directory for crashes, logs, cases
#=>   |  -h       Required, the host to send the fuzz case as host:port
{% endhighlight %}

If you see the newt help output as shown above without any errors, you should be good to go. 

## Collecting a corpus

Perhaps the most critical step in achieving a successful fuzz run is the collection of a corpus. These are the seed files that will be mutated by the fuzzer and fed into the target program. The more seed files, and the more these file differ, the better your run will be as that is the key to attaining the highest amount of code coverage in the subject binary. There are plenty of great guides available for choosing an effective corpus, however I will briefly describe the process I typically follow. 
Depending on the format you're working with, I find that your own machine is typically a good place to begin the search. In this guide we will be fuzzing PDFs, of which there are probably many on your drive. A quick look at my own Linux install reveals many documentation PDFs in a variety of languages utilizing a fairly wide array of features offered by the specification. Not a bad start for just a couple commands. 

{% highlight bash %}
mkdir seeds
cp `sudo find / -name *.pdf 2>/dev/null` seeds/
{% endhighlight %}

Next, I usually turn to Google, which offers us a handy operator to search by file type. A typical query might look like `filetype:pdf site:*.ru`. You'll notice in this example I restricted the search to Russian domains. The reason for this is to collect PDFs written in the Cyrillic alphabet. You can(and should) of course do this with any language. I find this helps to collect a corpus with more interesting inputs. Remember, we're trying to find PDFs that will trigger as many different functions in the target program as possible. 

At this point you are probably wondering how many inputs you should collect. The truth is the more the better, but you'll have to decide for yourself how much time you're willing to spend on this step. Since the point of newt is to get fuzzing fast, I typically don't collect more than a few hundred inputs, personally. 

## Starting an autofuzz run

If you're happy with your corpus, then you should be all set for the fun part: your first fuzz run with newt. Let's jump right in. 

{% highlight bash %}
mkdir out
./newt.js autofuzz -f 32 -s okular -m gdb -i seeds -o out -k 2
{% endhighlight %}

You should be off to the races. I'll briefly describe what's going on here. 

**-f** is the fuzz factor, which controls how mutated inputs should be. It essentially translates to fuzzing 1/-f bytes in the input file. In this example, we'll fuzz around 1/32 bytes. The lower the value, the more mutated the generated case will be. I typically fuzz with this value anywhere from 16-48. For programs particularly sensitive to file changes, you may need to increase this number quite a bit. On the other end of the spectrum, I find anything lower than about 8 tends to mangle files so badly the target rejects most cases without attempting to parse them. 

**-s** is the subject binary, with any necessary arguments. Unfortunately, these can only be arguments that do not use hyphens as newt's arguments parser uses this to denote its next flag. One way I get around this is to make a new alias with any arguments the target needs, and then use that as the argument to -s. Not ideal, but it seems to work fine for most things. Newt expects the program to open the case when fed from the command line, so in this example the command newt will run is `okular <case.pdf>`.

**-m** is the monitor mode. This argument tells newt's process manager how to monitor for crashes. Supported values are gdb, or asan. Monitoring with gdb requires the [exploitable module](https://github.com/jfoote/exploitable) by jfoote as it is used to triage crashes caught by gdb. Asan mode requires that the target binary be instrumented with Address Sanitizer. 

**-o** is the output folder for your fuzz run. In it, you will find newt's run log, a cases directory, where any cases that caused crashes will be saved, and a crash directory which will contain output from gdb or asan with more information about any observed crashes. 

**-k** is the time in seconds to wait before closing the program and moving on to the next case. This is what makes newt work so well with GUI programs. In this case it is set to 2 seconds, which is plenty of time to open and parse most PDFs. That is to say if the case is going to crash the reader, it will have done so by 2 seconds, at least on my machine. This argument is optional -- if you have a program that closes after the case has been analyzed then you can omit it.

Let the fuzzer run for as long as you wish, I usually let it do its thing overnight and come back and check the crashes directory in the morning. You can also run more than one instance of newt at once by creating multiple output directories and sharing the seeds directory. With a little luck, you'll have a few interesting crash cases to examine further. 

If you run into any issues or have questions about newt, feel free to contact me at chase [at] wreet [dot] xyz. Happy hunting! 
