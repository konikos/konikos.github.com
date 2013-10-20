---
layout: post
title:  "Bash job control"
date:   2013-01-20 04:03:00+02:00
description: Running multiple commands at once using Bash.
---

## Basics
Bash mantains a list of all currently running processes started by it. Each such process is called a job and can be run, either in the foreground or in the background. Foreground jobs are processes that are run by simply typing the command and, in most cases, take up the input and output of the terminal:

{% highlight console %}
$ echo Hello
Hello
{% endhighlight %}

On the other hand, background jobs run in the background (duh!) allowing Bash to mantain control of the terminal input and output. You can run a command in the background by appending a `&` to it:

{% highlight console %}
$ cat BIGFILE | grep 'Text' >results.txt &
[1] 1123
$ 
{% endhighlight %}

A more complex example, which runs multiple commands could be monitoring a process until it is finished and then send yourself a mail:

{% highlight console %}
$ (while kill -0 12345 &>/dev/null; do sleep 5m; done; mail -s "Process with PID 12345 finished!" <<<"Done" me@example.com) &
[1] 2468
$
{% endhighlight %}

Parentheses are needed, otherwise just the `mail` command would go to the background, sending the mail immediately, leaving the while loop working in the foreground. By parenthesizing, a new subshell is created, which runs the two commands and then the whole subshell is sent to the background.

To get a list of all currently running jobs, you can use the `jobs` Bash builtin:

{% highlight console %}
$ cat BIGFILE | grep 'Text' >results.txt &
[1] 1123
$ jobs
[1]+  Running        cat BIGFILE | grep 'Text' >results.txt
{% endhighlight %}
	
You can kill a job by using the `kill` builtin and referring to the job using the `%N` notation, where `N` is the id of the job:

{% highlight console %}
$ cat BIGFILE | grep 'Text' >results.txt &
[1] 1123
$ jobs
[1]+  running              cat bigfile | grep 'text' >results.txt
$ kill %1
$ echo Doing stuff
Doing stuff
[1]+ Terminated            cat BIGFILE | grep 'Text' >results.txt
{% endhighlight %}

As you can see, if the command terminates, or otherwise changes its running status, Bash will report it. Bash is pretty discreet, so it will only notify you the next time it presents you with the prompt.


What if you are already running a command that takes too long to finish and you want to send it the background to play some `nethack`? You can stop the foreground job temporarily by typing the suspend character, which usually is `^Z`, that is `Ctrl-Z`, and then send it to the background using the `bg` builtin.

{% highlight console %}
$ cat BIGFILE >duplicate.txt
^Z
[1]+  Stopped            cat BIGFILE >duplicate.txt
$ jobs
[1]+  Stopped            cat BIGFILE >duplicate.txt
$ bg %1
[1]+ cat BIGFILE >duplicate.txt
$ jobs
[1]+  Running            cat BIGFILE >duplicate.txt
{% endhighlight %}
	
You can ommit the `%1` to send to the background the current job, which is the last stopped job or the last job started in the background. So to quickly send a job to the background, all you have to do is `^Z` `bg`. By the way, using `cat` to copy a file is obviously a really bad idea, please don't do it.

To bring a job to the foreground, you can use the `fg` builtin.

{% highlight console %}
$ cat BIGFILE >duplicate.txt &
[1] 2357
$ fg %1
{% endhighlight %}

As with the `bg` command, `fg` can be used without arguments to bring to the foreground the current job. 

Bash provides shortcuts for both `fg` and `bg`, so instead of `fg %N`, just `%N` can be used. Similarly, `%N &` is a synonym for `bg %N`.


## Keep them running
So you played `nethack`, seeked in vain for hours for the Amulet of Yendor, but the command still hasn't finished running. You are connected to the computer over SSH, so you want to close the connection and get some sleep. If you simply exit the shell, any jobs running in the background will be terminated. This happens because, when Bash is terminating, it sends a SIGHUP signal to all of its running jobs. If left unhandled, this signal terminates the process. What you can do is instruct Bash to ignore a particular job and not send to it the SIGHUP signal. You can do it by running the `disown` builtin, which also removes completely the process from the job list.

{% highlight console %}
$ jobs
[1]+  Running            cat BIGFILE >duplicate.txt
$ disown %1
$ jobs
$
{% endhighlight %}

To keep the job in the list, but at the same time prevent Bash from sending it a SIGHUP, use the `-h` switch with ` disown`: 

{% highlight console %}
$ jobs
[1]+  Running            cat BIGFILE >duplicate.txt
$ disown -h %1
$ jobs
[1]+  Running            cat BIGFILE >duplicate.txt
$
{% endhighlight %}


If you know from the beggining that a command will take a long time, you can use the handy `nohup` utility to make it ignore SIGHUP signals, so that you don't have to disown it before exiting Bash.

{% highlight console %}
$ nohup cat BIGFILE >duplicate.txt &
[1] 3145
{% endhighlight %}

This way, the program will not terminate when you exit Bash. `nohup` obviously will only work with programs, not builtin commands, loops, the `;` command separator or other Bash constructs.

To prevent Bash sending the SIGHUP signal to any process at all, the `shopt` option `huponexit` can be used. To enable it and instruct Bash to send SIGHUP signals, run `shopt -s huponexit`. To disable it, run `shopt -u huponexit`. Note that the SIGHUP signal is not sent to any job, background or foreground. So having it disabled may sound like a good idea to have `huponexit` disabled, but it may lead to orpaned processes.

## Input and output of background jobs
The operating system has the notion of the current terminal process group. Members of this process group receive keyboard-generated signals such as SIGINT (usually assigned to `^C`). Foreground processes are members of this process group, while background processes are not and as such, they are not affected by keyboard-generated signals.

Only foreground processes are allowed to read from or write to the terminal. Background processes that attempt to read from or write to the terminal, receive the SIGTTIN or SIGTTOU signal respectively and by default they are supspended. Thus if we try:

{% highlight console %}
$ cat &
$ echo Doing other stuff
Doing other stuff
[1]+  Stopped        cat
$
{% endhighlight %}

`cat` is suspended. This is because it tried to read from the terminal while it was in the background and as a result it got a SIGTTIN signal which stopped the process. 

Even though background process are not permitted to read from the terminal, you can allow them to write to the terminal by changing the tostop stty setting:

{% highlight console %}
$ stty tostop
$ echo 2 &
[1] 13610
$ 
[1]+  Stopped      echo 2
$ stty -tostop
$ echo 2 &
[2] 1827
2
$
[2]+  Done         echo 2
{% endhighlight %}

Here, after setting tostop using `stty tostop`, `echo` can not produce any output while being in the background. On the other hand when the tostop setting is unset, `echo` does produce output and then terminates. On many systems the tostop setting is turned off by default, so you may notice that, without messing with `stty`, background processes can write to the terminal.

## References
Everything you may need is in the Bash manual. It may seem huge (and it actually is) but it is well written and everything about Bash, including job control, is described there. You can find it online [here](http://www.gnu.org/software/bash/manual/bashref.html).
