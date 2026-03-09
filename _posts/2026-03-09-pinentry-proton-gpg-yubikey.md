---
layout: single
title: "Pinentry for Proton Pass: GPG with YubiKey"
categories: [dev]
tags: [gpg, yubikey, protonpass, golang]
---

I use a [YubiKey](https://www.yubico.com/products/yubikey/) for GPG and SSH, and store all my credentials in [Proton Pass](https://proton.me/pass). For a while these things lived in parallel, mostly ignoring each other. Every GPG operation or SSH connection would pop up a `pinentry-mac` dialog asking for my passphrase. Fine, but it meant my passphrase existed in two places: the vault, and my memory. Every manual entry is a chance for a typo, a misremembered character, or a distraction at the wrong moment. The vault is the source of truth. It should be the only thing that knows the passphrase.

When Proton released [`pass-cli`](https://protonpass.github.io/pass-cli/), a proper command line interface for Proton Pass, I saw the opportunity to close that gap. The result is [`pinentry-proton`](https://github.com/damoun/pinentry-proton), a small binary I wrote that plugs into the GPG passphrase flow and fetches credentials from Proton Pass automatically.

## Prerequisites

- A [YubiKey](https://www.yubico.com/products/yubikey/) already configured for GPG (or SSH keys with passphrases)
- A [Proton Pass](https://proton.me/pass) account
- Go 1.21+ to build `pinentry-proton`
- Homebrew or `curl` for installing `pass-cli`

## The Pinentry Protocol

**Note:** You don't need to understand the protocol details to use this. Just know that `pinentry-proton` acts as an intermediary between `gpg-agent` and Proton Pass.
{: .notice--info}

For those curious: [`gpg-agent`](https://www.gnupg.org/documentation/manuals/gnupg/Invoking-GPG_002dAGENT.html) does not prompt for passphrases directly. It delegates to an external program called [`pinentry`](https://gnupg.org/related_software/pinentry/), communicating over the [Assuan protocol](https://gnupg.org/documentation/manuals/assuan/) (optional reading). The agent sends `GETPIN`, the pinentry returns the passphrase, done. Since `pinentry` is just a binary, you can replace it with anything that speaks the protocol.

`gpg-agent` also doubles as an [SSH agent](https://www.gnupg.org/documentation/manuals/gnupg/Agent-SSH-Support.html), handling SSH key passphrases through the exact same mechanism. So `pinentry-proton` covers both GPG and SSH in one shot.

Here is the complete flow:

```mermaid
flowchart LR
    A([gpg-agent]):::agent -->|GETPIN| B([pinentry-proton]):::binary
    B -->|item view| C([pass-cli]):::cli
    C -->|fetch| D([Proton Pass]):::vault
    D -.->|passphrase| C
    C -.->|passphrase| B
    B -.->|passphrase| A

    classDef agent  fill:#1e2a3a,stroke:#5b9bd5,color:#c8dff5
    classDef binary fill:#1a2a1e,stroke:#57a85a,color:#c8e6c9
    classDef cli    fill:#1e1a2e,stroke:#6d4aff,color:#d9ccff
    classDef vault  fill:#1e1a2e,stroke:#6d4aff,color:#d9ccff

    linkStyle 0,1,2 stroke:#666,stroke-width:1.5px
    linkStyle 3,4,5 stroke:#6d4aff,stroke-width:1.5px,stroke-dasharray:5
```

## Installing pass-cli

Full installation docs at [protonpass.github.io/pass-cli](https://protonpass.github.io/pass-cli/get-started/installation/). On macOS:

```bash
brew install protonpass/tap/pass-cli
```

On Linux:

```bash
curl -fsSL https://proton.me/download/pass-cli/install.sh | bash
```

Authenticate with your Proton account:

```bash
pass-cli login
```

**Note:** By default, `pass-cli` stores your session key in the system keychain. On a headless machine, set `PROTON_PASS_KEY_PROVIDER=fs` before running `auth login`. See the [configuration docs](https://protonpass.github.io/pass-cli/get-started/configuration/#2-filesystem-storage) for details.
{: .notice--info}

Confirm fetching works before going further:

```bash
$ pass-cli item view "pass://Personal/GPG Key/password"
your-secure-passphrase
```

If that returns your passphrase, the plumbing is in place. If `pass-cli` is not authenticated, you will see an error like `session not found`. Run `pass-cli login` again to restore it.

## Installing pinentry-proton

```bash
git clone https://github.com/damoun/pinentry-proton.git
cd pinentry-proton
make build
sudo make install
```

The binary ends up at `/usr/local/bin/pinentry-proton`.

## Configuration

`pinentry-proton` reads from `~/.config/pinentry-proton/config.yaml`. At minimum, set a `default_item` pointing to your GPG passphrase:

```yaml
default_item: "pass://Personal/GPG Key/password"
```

The URI format is `pass://VAULT_NAME/ITEM_TITLE/FIELD`. The last segment is the field name, so `/password` for the main password field, or any custom field name you have set.

For multiple keys, define mappings. `pinentry-proton` matches against metadata that `gpg-agent` sends with the `GETPIN` request (key description, keyinfo string):

```yaml
default_item: "pass://Personal/GPG Key/password"
mappings:
  - name: "Work GPG"
    item: "pass://Work/GPG Key/password"
    match:
      description: "work"
  - name: "YubiKey PIN"
    item: "pass://Personal/YubiKey PIN/password"
    match:
      description: "yubikey"
```

First match wins, falling back to `default_item`.

## Configuring gpg-agent

Point `gpg-agent` at the new binary in `~/.gnupg/gpg-agent.conf`:

```
pinentry-program /usr/local/bin/pinentry-proton
```

Restart the agent:

```bash
gpgconf --kill gpg-agent
```

The next GPG or SSH operation will go through `pinentry-proton`. No dialog, no keyboard input.

## YubiKey Considerations

With a YubiKey, `gpg-agent` asks for the card PIN rather than a software passphrase. `pinentry-proton` handles this the same way. Store the PIN in Proton Pass and reference it in the config.

**Warning:** The YubiKey PIN retry counter defaults to 3 attempts. If the vault entry is wrong, `gpg-agent` will keep submitting it until the counter hits zero and the card locks. Recovering from a locked card requires the Admin PIN. Run `gpg --card-edit`, then `admin` and `passwd` to reset it, as documented in the [YubiKey OpenPGP guide](https://support.yubico.com/hc/en-us/articles/360013790259-Using-Your-YubiKey-with-OpenPGP). Always verify the PIN manually before enabling automatic fetch.
{: .notice--danger}

```bash
$ pass-cli item view "pass://Personal/YubiKey PIN/password"
123456
```

Get that right first, then let the automation handle it.

## Debugging

Test `pinentry-proton` directly by simulating what `gpg-agent` sends:

```bash
$ pinentry-proton
GETPIN
D your-secure-passphrase
OK
```

Common failure modes:

**`pass-cli` is not authenticated**
You will see `ERR` returned from `pinentry-proton`. Run `pass-cli login` to restore the session.

**Wrong URI format**
The most common mistakes are a missing `/password` at the end, a wrong vault name, or a typo in the item title. Test the URI directly with `pass-cli item view` before putting it in the config.

**`gpg-agent` not picking up the config change**
Verify the agent is using the right binary:

```bash
gpgconf --list-options gpg-agent | grep pinentry
```

Make sure `gpg-agent.conf` has an absolute path to the binary. Kill the agent and let any `gpg` command relaunch it.

For more detail, enable debug logging:

```bash
PINENTRY_PROTON_DEBUG=1 pinentry-proton
```

## Quick Test

After setup, verify everything works end to end:

```bash
gpg --sign /dev/null
```

If your YubiKey blinks and no dialog appears, you are done.

If you give it a try, feel free to open an issue or a PR on the [GitHub repository](https://github.com/damoun/pinentry-proton). Especially interested in feedback from Linux users and anyone running non-standard vault structures.
