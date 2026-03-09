---
layout: single
title: "Pinentry for Proton Mail Bridge: GPG with YubiKey"
categories: [ops]
tags: [gpg, yubikey, protonpass, security]
---

I use [Proton Mail Bridge](https://proton.me/mail/bridge) to access my Proton Mail account through a standard email client. It works well, but it stores your Bridge credentials encrypted with GPG — and if you have a YubiKey configured as your GPG key, the interaction between Bridge, GPG agent, and pinentry gets interesting quickly.

The symptom I kept hitting: Bridge would prompt for my GPG passphrase, I'd type it, the YubiKey would blink, and then Bridge would fail to decrypt. The actual error was buried in logs. The root cause was pinentry.

## What Pinentry Does

GPG doesn't handle passphrase prompts directly. Instead it delegates to a separate program called `pinentry`, which is responsible for displaying the prompt and returning the passphrase to the agent. This separation exists so the same GPG agent can work in headless environments (SSH sessions), terminal environments, and GUI environments.

The problem is that there are multiple pinentry programs: `pinentry-curses`, `pinentry-gtk-2`, `pinentry-mac`, `pinentry-qt`, and others. The right one depends on your environment, and if GPG picks the wrong one, the dialog either doesn't appear or appears somewhere you can't see it — and the decryption silently fails.

## The Proton Mail Bridge Setup

Proton Mail Bridge on macOS is a GUI application. It runs in the background and handles SMTP/IMAP translation. When it needs to decrypt its stored session token, it calls GPG agent, which calls pinentry.

The issue is that Bridge is a GUI app without a terminal, so pinentry-curses can't work. And if `pinentry-mac` isn't set as the default, you get a broken prompt.

First, check what pinentry program your GPG agent is configured to use:

```bash
cat ~/.gnupg/gpg-agent.conf
```

It should have:

```
pinentry-program /opt/homebrew/bin/pinentry-mac
```

On Apple Silicon, Homebrew installs to `/opt/homebrew`. On Intel Macs it's `/usr/local`. Get the exact path with `which pinentry-mac`.

After changing `gpg-agent.conf`, reload the agent:

```bash
gpgconf --kill gpg-agent
gpgconf --launch gpg-agent
```

## YubiKey-Specific Gotchas

With a YubiKey, there's an additional layer. The YubiKey holds the actual private key — the local GPG agent has a stub that tells it to delegate to the card. When you decrypt something:

1. GPG agent detects the key is on a card
2. It calls pinentry to get the card PIN (not a passphrase)
3. Pinentry shows the "Enter PIN for YubiKey" prompt
4. You enter the PIN, the YubiKey performs the decryption

The PIN prompt for a YubiKey is different from a passphrase prompt for a key stored on disk. The timeout behavior is also different: the GPG agent caches passphrases for software keys, but YubiKey PIN caching depends on the agent's `default-cache-ttl-ssh` setting and the card's own PIN retry counter.

One thing that tripped me up: if you enter the wrong PIN three times, the YubiKey locks the key and requires the Admin PIN to unlock. This is a hardware limit, not a software one. `pinentry-mac` doesn't warn you about the retry count.

## Proton Pass Integration

There's a newer option I've started using: [pinentry-protonpass](https://github.com/nicholasgasior/pinentry-protonpass), a pinentry implementation that reads the passphrase from Proton Pass (the password manager) instead of prompting interactively.

The concept: instead of a dialog box, pinentry looks up the passphrase from your password vault. For automated workflows — scripts, background services like Bridge — this is cleaner than relying on a GUI prompt.

Setup requires adding the passphrase to Proton Pass under a specific path that the pinentry program knows to look for, and pointing `gpg-agent.conf` at the binary:

```
pinentry-program /usr/local/bin/pinentry-protonpass
```

For YubiKey specifically this is more limited since you're entering a PIN not a passphrase, but for software keys it works well.

## Debugging Pinentry Issues

If you're stuck, the fastest debug approach is to test pinentry directly from the terminal:

```bash
pinentry-mac
# Then type:
GETPIN
```

If you see `D <blank>` returned, the dialog appeared somewhere invisible. If you see a dialog appear and can type into it, pinentry-mac itself works, and the issue is in how GPG agent invokes it.

Also check the agent log:

```bash
tail -f ~/.gnupg/gpg-agent-log
```

Enable logging by adding `debug-level basic` and `log-file ~/.gnupg/gpg-agent-log` to `gpg-agent.conf`.

The whole setup — GPG agent + YubiKey + pinentry + Bridge — has a lot of moving parts that each have their own configuration. Once it works it's solid, but diagnosing breakage requires knowing which layer failed.
