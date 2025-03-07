# YubiKey SSH Agent

This project defines an SSH Agent specifically tailored for use with
YubiKeys.

The canonical home for this project is
https://github.com/indygreg/yubikey-ssh-agent.

## Usage

First, start up the agent:

    $ yubikey-ssh-agent --socket /tmp/yubikey-ssh.sock

Then, tell SSH how to use it:

    $ export SSH_AUTH_SOCK=/tmp/yubikey-ssh.sock

Then perform an SSH operation needing the private key on your YubiKey:

    $ ssh git@github.com

## Features

The `yubikey-ssh-agent` process provides a minimal SSH agent daemon
that interfaces directly with attached YubiKeys to service requests
for public key lookups and cryptographic signing operations.

The process provides a minimal GUI displaying current state and
provides a mechanism for inputting the PIN to unlock the YubiKey.

System notifications are displayed when the YubiKey needs to be
unlocked by entering a PIN.

## Advantages Over Normal SSH Agent

This tool was born because out of the author's frustration with the user
experience when using YubiKeys with OpenSSH using the default OpenSSH
agent (`ssh-agent`) and `libykcs11`.

When you use the default OpenSSH SSH agent + `libykcs11`:

1. `ssh-agent` spawns a `ssh-pkcs11-helper` process.
2. `ssh-pkcs11-helper` loads `libykcs11.{so,dylib,dll}`.
3. When `ssh-agent` receives a message requesting interfacing with the
   YubiKey, it calls into APIs in `libykcs11`, which speaks to the
   YubiKey.
4. Results from `libykcs11` are relayed back to `ssh`.

A common problem is that `libykcs11` will lose contact with the YubiKey
or your cached PIN expires due to a timeout. What happens in these
scenarios is `ssh-agent` thinks that no YubiKey keys are available
and tells `ssh` there are no keys. `ssh` summarily tries to
authenticate without knowledge of the YubiKey keys. And this often
fails with a `Permission denied` message because the client didn't
actually present any public keys!

Or a variant of this is that `ssh-agent` advertises the YubiKey-hosted
key but when it attempts to use the key it fails because the YubiKey is
locked (a PIN is required). This also often materializes as a nebulous
and hard-to-debug `Permission denied` error.

**Unlike the default `ssh-agent` + `libykcs11` behavior, this agent
won't fail SSH client operations because the YubiKey is locked, the agent
lost a connection with the YubiKey, or the agent's cached PIN has expired.
Instead, this agent recognizes when a key is locked and prompts the user
to unlock it, before failing the SSH operation.**

This SSH agent makes the assumption that the YubiKey is the provider of
SSH keys. Therefore, when there is a request for available keys or a
signature request, it can be very vocal about raising an error (through
its own GUI or OS notifications) when user interaction is needed. For
example, if SSH wants to perform a cryptographic signature but the YubiKey
is locked, this agent will show you a system notification that the YubiKey
PIN needs to be entered and the SSH agent will wait for you to unlock the
YubiKey before failing the SSH attempt.

### Security Advantages

This agent doesn't support adding keys. This agent doesn't (yet) support
caching the YubiKey PIN or management key.

There are no secrets lingering in memory that can easily be extracted by
a user on the same machine. (If someone accesses the process at just the
right time they could acquire the PIN, however.)

The main threat model for this SSH agent is an unwanted client requesting
signing operations. This threat model exists for all SSH agent implementations.
For the ultra paranoid, you'll want to set a PIN protection policy on the
YubiKey to require a PIN or touch for every operation. Without such a policy,
multiple signing operations may be performed from a single unlock/touch
and anyone with access to the SSH agent could effectively use the private
key on the YubiKey.

## State of Project

This project is still very alpha. The graphical UI in particular is
very crude and in need of a lot of work.

Please file issues or just contribute pull requests to improve things.

Only macOS is well tested. Windows doesn't currently build due to
https://github.com/sekey/ssh-agent.rs not compiling on Windows (this
is a very fixable problem).
