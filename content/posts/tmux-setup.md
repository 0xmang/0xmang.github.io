---
title: "About"
date: 2026-04-12
draft: false
ShowReadingTime: false
ShowBreadCrumbs: true
ShowPostNavLinks: false
ShowToc: false
cover:
  image: "/images/windows.jpg"
  alt: "TMUX setup"
  caption: "Documenting cloud, security, automation, and technical projects."
---

# Using Tmux instead of paid SSH Clients

# Introduction

Simply, Tmux is a ***free*** and open-source terminal multiplexer for Unix-based systems. It does support multiple windows, panes and detachable sessions. It allows efficient and smart way to manage independent sessions, all in the same viewable terminal.

Tmux is really important and life-saver for those who spend much of their working hours connecting to various servers and doing command line tasks, e.g. administrators, developers, integrators, and operation engineers.

Tmux have hundreds of keyboard shortcuts that speeds up things and enable efficient and smart ways of working.

In this tutorial, I will explain how to set up Tmux, and Tmuxinator, which is basically a ruby gem that builds on top of tmux to simplify managing complex tmux sessions. With Tmuxinator you can build your project (set of tmux windows/sessions/panes) using yaml configuration files which can be used as templates, easily shared, and upgradable.

# Setup

## 1. Set up WSL

[Here](https://learn.microsoft.com/en-us/windows/wsl/install) you can set up WSL on  your workstation. Make sure that you set WSL version to 2.

Install Ubuntu on WSL2 

Use [this](https://canonical-ubuntu-wsl.readthedocs-hosted.com/en/latest/guides/install-ubuntu-wsl2/) link to install Ubuntu on your windows workstation.

## Install Tmux

Update the setup Ubuntu distro.

```bash
sudo apt update
sudo apt upgrade -y
```

Setup Tmux

```bash
sudo apt install tmux
```

Now you can launch Tmux, using `tmux` command.

Set up Tmuxinator

```bash
sudo apt-get install -y tmuxinator
```

# Sample Setup

Below are steps to set up your own tmuxinator project

Assuming you always access server X, and server Y. On each, once you login, you would check filesystem size, pods status of a cluster, and tail a log file

For the purpose of the completeness of this example, there will be steps involved to configure tmux itself to meet our intentions.

**Some important files/directories involved:**

1- `.tmux.conf` This is tmux configuration file. Usually you create this on your home directory or some `.config` directory under it.

2- `project.yaml` : This will be the file that we will be generated (and we’ll modify it) once you create a new tmuxinator project

3- `~/.config/tmuxinator` This is the default directory where new tmuxinator project files, i.e. yamls are stored

**Modifications in .tmux.conf** 

These lines, I modified/added, are totally optional, but I thought to include

`set-option -g mouse on` : This will enable mouse usage in tmux; very helpful

Default Tmux key-prefix is Ctrl-b, which, to me, felt a bit too much, so I wanted to change it to Ctrl-Space, below 3 lines does the thing

`unbind C-b` 

`set-option -g prefix C-Space`
`bind-key C-Space send-prefix`

`set-option -g window-active-style 'fg=lightgreen'`Set the font of the active window/pane to lightgreen

Below 3 lines customize pane borders. For each pane, it sets the fore- and back-ground colors, plus pane title written in the border

```bash
set -g pane-border-status bottom
set -g pane-border-format "#{pane_index} #{pane_title}"
set-option -g pane-border-format "#{?#{==:#{pane_title},Main},#[fg=white bg=red],#{?#{==:#{pane_title},Disk_Space},#[fg=black bg=yellow],#{?#{==:#{pane_title},Pods},#[fg=white bg=green],#{?#{==:#{pane_title},tail},#[fg=white bg=blue],#{?#{==:#{pane_title},WhoamI},#[fg=white bg=brown],#[default]}}}}}#{pane_index} #{pane_title}"
```

Now is time to initialize our sample project

```bash
tmuxinator new First
```

`First.yaml` will be create in `~/.config/tmuxinator/`

The main section is the last one, called `windows` Before it, there are many parameters that can be configured and could be very useful, say to automatically enable logging of sessions at start up. You ca google these parameters should you want to customize more.

First.yaml:

```bash
windows:
  - Server X:
      layout: tiled
      panes:
        - Main:
          - echo 'Main X pane'
        - Filesystem:
          - df -kh
        - Pods:
          - kubectl get pods -A
        - Tail:
          - tail -10f /var/log/dpkg.log
  - Server Y:
      layout: tiled
      panes:
        - Main:
          - echo 'Main Y pane'
        - Filesystem:
          - df -kh
        - Pods:
          - kubectl get pods -A
        - Tail:
          - tail -10f /var/log/dpkg.log
```

To run it:

```bash
tmuxinator First
```

![Untitled](/images/Untitled.png)

By default, pane titles will default to hostname (redacted), and this can be changed via below command, for each pane you select do:

Ctrl+Space `:select-pane -T <title>`

![Untitled](/images/Untitled%201.png)

And it will change to:

![Untitled](/images/Untitled%202.png)