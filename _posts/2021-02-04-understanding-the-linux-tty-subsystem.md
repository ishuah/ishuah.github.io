---
layout: post
title: Understanding The Linux TTY Subsystem
description: A deep dive into the history, legacy, and inner workings of the Linux TTY subsystem.
date: 2021-02-04 07:28:22
image: '/images/26.jpg'
tags: [linux, tty]
---

_Photo by <a href="https://unsplash.com/@_imkiran?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Sai Kiran Anagani</a> on <a href="https://unsplash.com/s/photos/linux?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a>_

TTY stands for TeleTYpe. If you Google the word teletype, a picture of a device that looks like a typewriter shows up. How did a typewriter become an essential part of the Linux operating system?

## how it all started
The teletype came about through a series of innovations around message transmission on electric channels. It has a rich history going back to the 1840s. Several innovations and collaborations led to the development of the [Telex exchange network](https://en.wikipedia.org/wiki/Telex) in the late 1920s. Telex eventually grew to over 100,000 connections worldwide and was vital in global communication post World War II.

Meanwhile, computer technology was progressing. Earlier computers could only run one program at a time but in the 1960s, multiprogramming computers appeared on the market. These computers could interact with users in realtime via a command-line interface. There was suddenly a need for input and output devices. Instead of building new I/O machines, pragmatic engineers reused existing teletypes. Teletypes were already on the market, and they fit the use case perfectly as physical terminals for mainframe computers.

Users could now type commands on the teletype and receive the computer output via punched tape. Later versions of the teletype were completely electronic and utilized electronic screens. Users could move the cursor and clear the screen, features unavailable on printed paper teletypes.

## the teletype as a physical terminal
<figure><img src="/images/teletype-terminal.jpg"><figcaption>The Teletype terminal</figcaption></figure>

A physical line connects the teletype to the Universal Asynchronous Receiver and Transmitter on the computer. This physical line consists of two cables, an input and output cable. When a user types on the terminal, the input cable sends the keystrokes to the UART driver, which sends the keystrokes to the line discipline layer. The line discipline does three things:
* It echoes the keystrokes to the output cable (back to the terminal/teletype), so the user can see what they've typed 
* It manages the character buffer.
* It handles special editing commands (erase, backspace, clear line).

When the user presses the enter key, line discipline sends the buffered characters to the TTY driver, which passes the characters to the foreground process attached to the TTY.
The UART driver, line discipline, and the TTY driver make up the TTY device.

## terminal emulators
As technology improved, computers shrunk in size, and teletypes became cumbersome. Software emulated teletypes replaced physical teletypes. These terminal emulators work in the same way as their physical counterparts, the only difference being that there are no physical lines and UART connections. Examples of terminal emulators include xterm and the gnome-terminal.

<img src="/images/terminal-emulator.jpg"/>

## pseudo-terminals (PTY)
The whole TTY subsystem residing in the kernel made terminal interactions inflexible. The solution was to move terminal emulation to the userland, leaving line discipline and the TTY driver intact in the kernel. The PTY consists of two parts:
* The PTY master (PTMX) - attached to the terminal emulator
* The PTY slave  (PTS) - provides processes with an interface to the PTY master

<img src="/images/pseudo-terminal.jpg"/>

Note: the line discipline instance is not evoked when the TTY driver is sending user program output to PTY master.

## shell
So far, we've covered terminal emulators and pseudo-terminals. The shell is a program that resides in userland and manages user-computer interactions. Examples of shell programs include bash, zsh, and fish. 

## putting it all together
What happens when you open a terminal emulator on Linux? I'll give an example with my environment: I use gnome-terminal as my terminal emulator and zsh as my shell.
1. Gnome-terminal renders its UI on the video display.
2. It requests a PTY from the OS.
3. It launches a zsh subprocess.
4. It sets the stdin, stdout, and stderr of zsh to PTY slave.
5. It listens for keystroke events and sends the characters to PTY master.

<img src="/images/htop.png" />

Systemd is Linux's system and service manager. As shown above, systemd spawns a gnome-terminal subprocess, which in turn starts a zsh subprocess. The output above is from htop, which I ran from my terminal emulator. So, what happens when you run a command on a terminal emulator?
1. PTY master receives characters from gnome-terminal and passes them to the line discipline layer.
2. Line discipline layer buffers the characters as you type them, writing back to PTY master so that you can see what you're keying in.
3. When you press enter, the line discipline layer sends the characters to the TTY driver, which then passes the characters to the PTY slave.
4. zsh reads the characters "htop" and forks the process to run the htop program. The forked process has the same stdin stdout and stderr as zsh.
5. htop prints to stdout (PTY slave).
6. PTY slave passes the output to PTY master, which passes it on to gnome-terminal.
7. Gnome-terminal reads the output and redraws the UI. The read is a loop, any change in the htop output reflects on the display in real-time.

## conclusion
This was a brief introduction to the Linux TTY subsystem. Much of how it operates right now is influenced by technical decisions made over 60 years ago. Remarkable resilience.
If you have any remarks or feedback, please reach out on <a target="blank_" href="https://twitter.com/ishuah_">twitter!</a>

Technical references:
* [The TTY demystified by Linus Åkesson](http://www.linusakesson.net/programming/tty/)
* [Linux terminals, tty, pty and shell by Nicola Apicella](https://dev.to/napicella/linux-terminals-tty-pty-and-shell-192e)
* [What is a TTY on Linux? (and How to Use the tty Command) by Dave McKay](https://www.howtogeek.com/428174/what-is-a-tty-on-linux-and-how-to-use-the-tty-command/)