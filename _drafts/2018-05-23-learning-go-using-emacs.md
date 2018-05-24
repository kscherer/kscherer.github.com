---
layout: post
title: "Learning Go using Emacs"
category: Golang
tags: [golang, emacs, linux]
---

## Emacs

Working on my emacs config is always a good way to get
distracted. First I decided to focus on using company-mode for
autocomplete instead of auto-complete which never quite worked the way
I wanted.

First step is enable go-mode. Flycheck already supports several go
tools for syntax checking. Company mode requires an extra backend
called company-go.
