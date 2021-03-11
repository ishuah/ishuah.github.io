---
layout: post
title: Build A Simple Terminal Emulator In 100 Lines of Golang
description: A deep dive into the history, legacy, and inner workings of the Linux TTY subsystem.
date: 2021-03-10 06:13:57
image: '/images/27.jpg'
tags: [linux, tty, terminal-emulator]
---

_Photo by <a href="https://unsplash.com/@ngeshlew?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Lewis Ngugi</a> on <a href="/collections/1173243/terminal?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>_

In a previous [article](https://ishuah.com/2021/02/04/understanding-the-linux-tty-subsystem/), I wrote a brief introduction to the current state of the TTY Subsystem. This article builds on the concepts covered in that article, adding a practical understanding of how the TTY Subsystem works. We're building a simple [terminal emulator](https://en.wikipedia.org/wiki/Terminal_emulator) in Golang. This is installment is the second article in my 'terminal emulator' series. This post is the first of my three-part "terminal emulator" series.

## the user interface
The first thing we're going to build is the user interface. It's nothing fancy, just a triangle with legible text in it.
I'll use the [Fyne UI Toolkit](https://github.com/fyne-io/fyne). It's mature and reasonably [documented](https://developer.fyne.io/). 
Here's our first iteration:

{% highlight go %}
package main

import (
	"fyne.io/fyne/v2"
	"fyne.io/fyne/v2/app"
	"fyne.io/fyne/v2/layout"
	"fyne.io/fyne/v2/widget"
)

func main() {
	a := app.New()
	w := a.NewWindow("germ")

	ui := widget.NewTextGrid()       // Create a new TextGrid
	ui.SetText("I'm on a terminal!") // Set text to display

	// Create a new container with a wrapped layout
	// set the layout width to 420, height to 200
	w.SetContent(
		fyne.NewContainerWithLayout(
			layout.NewGridWrapLayout(fyne.NewSize(420, 200)),
			ui,
		),
	)

	w.ShowAndRun()

}
{% endhighlight %}
[Link to the gist](https://gist.github.com/ishuah/6b5b97131639c7ce410abb7b9caecec3)

The program above uses the Fyne UI API to render a text grid with the text "I'm on a terminal!".

<img src="/images/01-ui.png">

## pseudoterminal
The next step involves connecting to the TTY driver that lives in the kernel. We'll be using the [pseudoterminal](https://en.wikipedia.org/wiki/Pseudoterminal) for this task. Like the TTY driver, the pseudoterminal lives in the OS kernel. It consists of a pair of pseudo-devices, a pty master, and a pty slave. 

<figure><img src="/images/pseudo-terminal.jpg"><figcaption>The pty master and pty slave communicate with each other through the TTY driver</figcaption></figure>

The pty slave receives all its input from the pty master. It also sends all its output to pty master. The pty master sends keystrokes from the keyboard to the pty slave. It also prints output from the pty slave to the display.
I'll use [pty](https://github.com/creack/pty), a Go package for interfacing with Unix pseudoterminals.

{% highlight go %}
package main

import (
	"os"
	"os/exec"
	"time"

	"fyne.io/fyne/v2"
	"fyne.io/fyne/v2/app"
	"fyne.io/fyne/v2/layout"
	"fyne.io/fyne/v2/widget"
	"github.com/creack/pty"
)

func main() {
	a := app.New()
	w := a.NewWindow("germ")

	ui := widget.NewTextGrid()       // Create a new TextGrid

	// Start bash process
	c := exec.Command("/bin/bash") // @Line 22
	p, err := pty.Start(c)

	if err != nil {
		fyne.LogError("Failed to open pty", err)
		os.Exit(1)
	}

	defer c.Process.Kill() // @Line 30

	p.Write([]byte("ls\r")) // @Line 32
	time.Sleep(1 * time.Second)
	b := make([]byte, 1024)
	_, err = p.Read(b)
	if err != nil {
		fyne.LogError("Failed to read pty", err)
	}

	ui.SetText(string(b))
	// Create a new container with a wrapped layout
	// set the layout width to 420, height to 200
	w.SetContent(
		fyne.NewContainerWithLayout(
			layout.NewGridWrapLayout(fyne.NewSize(420, 200)),
			ui,
		),
	)

	w.ShowAndRun()

}
{% endhighlight %}
[Link to the Gist](https://gist.github.com/ishuah/e54a5445cf5ec0352915a508f1955bbd)

In the code above, lines 22-30 handle starting the bash process. We now have a pty master pointer p (line 23).

Line 32, `p.Write([]byte("ls\r"))`, writes the characters `ls` and the return carriage byte to the pty master. As explained in the diagram above, the pty master sends these characters to the pty slave. Simply put, we've sent a command to the bash process.

The program above results in a text grid with an unordered list of items in your current directory.

<img src="/images/02-pseudoterminal.png">

## input from keyboard
We're now going to read input from the keyboard and write to the pty master. The Fyne UI toolkit provides an effortless way of capturing keyboard input.

{% highlight go %}
package main

import (
	"os"
	"os/exec"
	"time"

	"fyne.io/fyne/v2"
	"fyne.io/fyne/v2/app"
	"fyne.io/fyne/v2/layout"
	"fyne.io/fyne/v2/widget"
	"github.com/creack/pty"
)

func main() {
	a := app.New()
	w := a.NewWindow("germ")

	ui := widget.NewTextGrid()       // Create a new TextGrid
	ui.SetText("I'm on a terminal!") // Set text to display

	c := exec.Command("/bin/bash")
	p, err := pty.Start(c)

	if err != nil {
		fyne.LogError("Failed to open pty", err)
		os.Exit(1)
	}

	defer c.Process.Kill()

	onTypedKey := func(e *fyne.KeyEvent) {
		if e.Name == fyne.KeyEnter || e.Name == fyne.KeyReturn {
			_, _ = p.Write([]byte{'\r'})
		}
	}

	onTypedRune := func(r rune) {
		_, _ = p.WriteString(string(r))
	}

	w.Canvas().SetOnTypedKey(onTypedKey)
	w.Canvas().SetOnTypedRune(onTypedRune)

	go func() {
		for {
			time.Sleep(1 * time.Second)
			b := make([]byte, 256)
			_, err = p.Read(b)
			if err != nil {
				fyne.LogError("Failed to read pty", err)
			}

			ui.SetText(string(b))
		}
	}()

	// Create a new container with a wrapped layout
	// set the layout width to 420, height to 200
	w.SetContent(
		fyne.NewContainerWithLayout(
			layout.NewGridWrapLayout(fyne.NewSize(420, 200)),
			ui,
		),
	)

	w.ShowAndRun()

}
{% endhighlight %}
[Link to the Gist](https://gist.github.com/ishuah/6e39845670b156ffae429b986d283d13)

SetOnTypedKey captures special keypress events and passes them to our callback function, onTypedKey. Special keys include the escape key, enter key, backspace, etc. Our callback function only handles one special keypress event, the enter keypress. 
SetOnTypedRune works very similarly to SetOnTypedKey, except instead of special keypress events, it captures character keypress events. The attached callback function onTypedRune writes the characters to the pty master.
I've also added a goroutine that reads from the pty master and writes to our UI text grid. There's no buffer or cursor management. The result is beautiful chaos.
When you run the program above and type a command, let's say 'ls -al.' You should see the UI update with the expected output. If you like throwing caution to the wind, run `ping 8.8.8.8`.

<figure>
	<img src="/images/htop-view-germ.png">
	<figcaption>Bash runs as a subprocess, spawned by our program. Bash receives the command `ping 8.8.8.8` and spawns a subprocess.</figcaption>
</figure>

We're not handling signals yet, so Ctrl-C will not stop the job. You'll have to terminate the terminal emulator.

## print to screen
So far, we can type commands on our terminal emulator, receive command output, and chaotically print to our UI. Let's make a few improvements on how we print to screen. We'll update our display goroutine to display the pty output history instead of just printing the last line on the screen. The pseudoterminal doesn't manage output history. We'll have to handle it ourselves using an output buffer.

{% highlight go %}
package main

import (
	"bufio"
	"io"
	"os"
	"os/exec"
	"time"

	"fyne.io/fyne/v2"
	"fyne.io/fyne/v2/app"
	"fyne.io/fyne/v2/layout"
	"fyne.io/fyne/v2/widget"
	"github.com/creack/pty"
)

// MaxBufferSize sets the size limit
// for our command output buffer.
const MaxBufferSize = 16

func main() {
	a := app.New()
	w := a.NewWindow("germ")

	ui := widget.NewTextGrid() // Create a new TextGrid

	os.Setenv("TERM", "dumb")
	c := exec.Command("/bin/bash")
	p, err := pty.Start(c)

	if err != nil {
		fyne.LogError("Failed to open pty", err)
		os.Exit(1)
	}

	defer c.Process.Kill()

	// Callback function that handles special keypresses
	onTypedKey := func(e *fyne.KeyEvent) {
		if e.Name == fyne.KeyEnter || e.Name == fyne.KeyReturn {
			_, _ = p.Write([]byte{'\r'})
		}
	}

	// Callback function that handles character keypresses
	onTypedRune := func(r rune) {
		_, _ = p.WriteString(string(r))
	}

	w.Canvas().SetOnTypedKey(onTypedKey)
	w.Canvas().SetOnTypedRune(onTypedRune)

	buffer := [][]rune{}
	reader := bufio.NewReader(p)

	// Goroutine that reads from pty
	go func() {
		line := []rune{}
		buffer = append(buffer, line)
		for {
			r, _, err := reader.ReadRune()

			if err != nil {
				if err == io.EOF {
					return
				}
				os.Exit(0)
			}

			line = append(line, r)
			buffer[len(buffer)-1] = line
			if r == '\n' { // @Line 72
				if len(buffer) > MaxBufferSize { // @Line 73 If the buffer is at capacity...
					buffer = buffer[1:] // ...pop the first line in the buffer
				}

				line = []rune{}
				buffer = append(buffer, line)
			} // @Line 79
		}
	}()

	// Goroutine that renders to UI
	go func() {
		for {
			time.Sleep(100 * time.Millisecond)
			ui.SetText("")
			var lines string
			for _, line := range buffer {
				lines = lines + string(line)
			}
			ui.SetText(string(lines))
		}
	}()

	// Create a new container with a wrapped layout
	// set the layout width to 900, height to 325
	w.SetContent(
		fyne.NewContainerWithLayout(
			layout.NewGridWrapLayout(fyne.NewSize(900, 325)),
			ui,
		),
	)
	w.ShowAndRun()
}

{% endhighlight %}
[Link to the Gist](https://gist.github.com/ishuah/fa78f31e3ec1cc3f84ffe0a25dd1cf17)

Our output buffer comes in the form of a slice of rune slices. I've declared a constant MaxBufferSize to limit buffer size because this is what sane people do. You're probably wondering why I used a slice instead of an array. We'll come to that in a bit.

In our previous iteration, we read from pty master in chunks of 256 bytes. I've updated our reader goroutine to use a [bufio.Reader](https://golang.org/pkg/bufio/#Reader). The bufio.Reader type has a method [ReadRune](https://golang.org/pkg/bufio/#Reader.ReadRune) that returns a single rune and its size in bytes. This method feeds our output buffer one rune at a time. 

Line 72 - 79 defines an if block that checks whether the read rune is a newline character. If it is, we know that we're about to append a new line. Before we do this, we first check whether the buffer is at capacity (line 73). If it is, we pop the first line to create space for the new line. MaxBufferSize is 16. This way, we always have a buffer of length 16 holding the last 16 lines of pty master output. 

The display functionality is now running on its goroutine. It redraws the display every 100 milliseconds. 

## conclusion
We now have a simple terminal emulator in ~~100~~ 106 lines of Go! Of course, there's still a long way to go before our tiny program can be called a functional terminal emulator. The next article in this series will cover special keys, signals, and the all-powerful cursor.