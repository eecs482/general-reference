# Tmux Remote Pair Programming
This guide aims to get you familiar with using `tmux`, a "terminal multiplexor" to pair program with others remotely. It is especially useful if your group prefers terminal-based text editors like `(n)vim`, `emacs` or `nano`. It is also useful to share access to multiple terminals.

## Using `tmux` in CAEN
Since CAEN is the only officially supported environment and remote access to a CAEN terminal is relatively straightforward, this guide will focus on pair programming on CAEN. Many of the steps are the same for local use as well, but you will have to enable SSH access to your computer, which usually involves setting up Port Forwarding on your router.

### Setup (one time)
While CAEN does have `tmux` installed by default, it is owned by root, which means you can't provide any config scripts. A much more flexible setup will be to get a universal appimage from the [tmux](https://github.com/tmux/tmux/releases) release page. You can copy/paste the following code to download the appimage directly to CAEN:
```
wget https://github.com/tmux/tmux/releases/download/3.1b/tmux-3.1b-x86_64.AppImage
```
Now that you have the app image downloaded, lets use `chmod` to make it executable
```
chmod +x tmux-3.1b-x86_64.AppImage
```

As a sanity check, run `./tmux-3.1b-x86_64.AppImage` and make sure it doesn't crash. If it works (it should look like a terminal with a green bar at the bottom), then press the control button and then "d" (while still holding control).

### Step 1: connect to the same CAEN machine
The host of the pair programming session should ssh into a CAEN terminal. Lets call this person Alice. Once Alice is connected to CAEN, they should run `hostname` to determine which CAEN server they are connected to. The output will likely be something like `caen-vnc-vmXX.engin.umich.edu` where each X is some number. The person who wants to pair with Alice, lets call them Bob, will then have to SSH directly to this machine by running `ssh uniqname@caen-vnc-vmXX.engin.umich.edu`, where `uniqname` is replaced with Bob's uniqname.


### Step 2: Create a sharable `tmux` session
Now, the host, Alice, should run `./tmux-3.1b-x86_64.AppImage -S /tmp/$(whoami) new -s shared` to create a shared `tmux` session. This will create a "socket file" of the path `/tmp/<uniqname of alice>`.

### Step 3: Bob attaches to Alice's session
Now, Bob can attach to Alice's session, and effectively plug in his keyboard to her terminal(s).
To do so, he should run `./tmux-3.1b-x86_64.AppImage -S /tmp/<Alice's uniqname> attach -t shared`

### Step 4: Profit
You'll want some way to ensure 2 or 3 of you don't try to type at the same time, as everyone is tied to the same session. This mirrors pair programming in real life, where there is only one keyboard.
Tmux is pretty powerful, allowing you to create multiple windows and multiple panes within those windows. This helps if you want one window for editing text, another for compiling/running it, and maybe a third to track notes in or edit some local scripts. You can find a quick tmux guide [here](https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/), and many more just like it (including useful customizations) online.

### Extra credit
* You can rename the AppImage file to just `tmux` if you want. I recommend doing so and putting it into the `~/.local/bin/` directory (creating it if it doesn't exist) and adding that directory to your PATH (via `export PATH="~/.local/bin:$PATH"` in your `~/.profile`). This way, you can just run `tmux ...` instead of `./tmux-3.1b ...`.
* Allow users to be independent. I haven't done this before, but [this tutorial](https://www.hamvocke.com/blog/remote-pair-programming-with-tmux/) suggests that instead of Alice using `-t shared` she should use `-t alice` and that bob should replace `-t shared` with `new -s bob -t alice`. This will allow them to share the content of their windows, but be connected to them independently. You'll still want to avoid typing at the same time to the same pane (as I imagine `bash`/`vim`/etc. won't understand that two different people are communicating to them at the same time), but this allows for everyone to get their own pane and work independently (say, writing test cases).
