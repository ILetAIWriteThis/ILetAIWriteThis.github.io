---
layout: post
title: "From Zero to Blog: My Dev Setup"
category: serious-stuff
topic: Dev Preparation
---

I wanted a blog. I didn't want to spend a week configuring things. Here's exactly what I did to go from a fresh machine to a live GitHub Pages site -- with AI doing the heavy lifting.

## Step 1: Clean WSL Slate

First things first -- nuke any old WSL install and start fresh. No leftover configs, no mystery packages from six months ago.

```
wsl --unregister Ubuntu
```

This wipes the entire Ubuntu instance. Everything. Gone. Then reinstall clean:

```
wsl --install Ubuntu
```

Fresh terminal, fresh start. Set your username and password when prompted and you're in.

## Step 2: SSH Keys for GitHub

You need SSH access to push code. No HTTPS password prompts, no tokens expiring at the worst possible moment.

Generate a new key:

```
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Hit enter through the prompts (or set a passphrase if you're disciplined like that). Then start the agent and add the key:

```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

Copy the public key:

```
cat ~/.ssh/id_ed25519.pub
```

Go to **GitHub > Settings > SSH and GPG keys > New SSH key**, paste it in, done. Test it:

```
ssh -T git@github.com
```

If you see "Hi username!", you're golden.

## Step 3: Install Claude Code

One line:

```
curl -fsSL https://claude.ai/install.sh | bash
```

That's it. Claude Code CLI is now available in your terminal. This is the tool that actually built the blog you're reading right now.

## Step 4: Let AI Build Your Blog

This is where it gets interesting. Instead of manually setting up Jekyll, picking a theme, writing CSS, and fighting with Liquid templates -- I just told Claude what I wanted:

- Dark mode
- Three categories (Thoughts, Serious Stuff, Non-Binding Stuff)
- Topic-based organization instead of date-based
- A search bar and filter pills

Claude generated the entire Jekyll structure: layouts, CSS, index page with filtering logic, example posts, the works. The theme you're looking at right now? Opus built it.

## The Full Flow

To recap, the entire process from nothing to published blog:

1. `wsl --unregister` + `wsl --install` -- clean Linux environment
2. `ssh-keygen` -- authentication sorted
3. `curl ... | bash` -- Claude Code installed
4. Tell AI what you want -- blog structure generated
5. `git push` -- live on GitHub Pages

No theme hunting. No Stack Overflow rabbit holes. No "works on my machine" debugging. Just describe what you want and ship it.

## Was It Worth It?

The whole setup took less time than it would have taken me to pick a Jekyll theme I didn't hate. And the result is exactly what I asked for -- not some template I'd have to fight with for the next year.

Sometimes the best dev preparation is knowing which parts to automate.
