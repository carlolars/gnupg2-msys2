
- [Using GnuPGv2 with MSYS2](#using-gnupgv2-with-msys2)
    - [Change GnuPG home (optional)](#change-gnupg-home-optional)
    - [Install GnuPG](#install-gnupg)
    - [Paths](#paths)
        - [Windows *Path*](#windows-path)
        - [MSYS2 *PATH*](#msys2-path)
    - [**gpg** and **mintty** needs **winpty**](#gpg-and-mintty-needs-winpty)
    - [Enter passphrase in the terminal instead of a popup window](#enter-passphrase-in-the-terminal-instead-of-a-popup-window)
- [SmartCard SSH Authentication in MSYS2](#smartcard-ssh-authentication-in-msys2)
- [.bash_profile](#bash-profile)
- [Problems](#problems)
    - ['Windows HELLO for Business' and SmartCard problem with GnuPG](#windows-hello-for-business-and-smartcard-problem-with-gnupg)

# Using GnuPGv2 with MSYS2
MSYS2 comes with version 1.4 of GnuPG and will (probably) never get version 2.

There are mingw32/64 packets that can be installed using pacman but they don't work at all, at least not the mingw64 version I have tested.

Instead I use the official build of GnuPG from www.gnupg.org (2.2.1), and it works well.

## Change GnuPG home (optional)
The default home directory used by GnuPG is `%APPDATA%/gnupg`.

If you want to use another path then you must create an environmental variable for your user so that both MSYS2 terminal and Windows terminal use the same home for GnuPG.

This is easiest to do from the command line using the `SETX` command, for example to set the GnuPG home to `H:\.gnupg` use the following command:
```bash
# From the Windows Terminal:
>setx GNUPGHOME H:\.gnupg
# or from the MSYS2 Terminal:
$ setx GNUPGHOME /h/.gnupg
```

## Install GnuPG
Download and install [Gpg4win](https://www.gpg4win.org/download.html). Gpg4win will install the same GnuPG as the *simple installer* from [GnuPG](www.gnupg.org) but with some additional GUI applications and more importantly a better pinentry.

## Paths
### Windows *Path*
The path to gnupg/bin is added last to the Windows *Path* during the installation, but if you already have the paths to MSYS2/Mingw32/Mingw64 bin directories (default C:\msys64\usr\bin, C:\msys64\mingw32\bin or C:\msys64\mingw64\bin) then you must make sure that the path to GnuPG/bin is *before* them:
```
...
C:\Program Files (x86)\gnupg\bin
C:\msys64\mingw64\bin
C:\msys64\usr\bin
```
If gnupg/bin is last then the `gpg` command in a Windows terminal will use the 1.4 version from usr\bin since it will be found first.

### MSYS2 *PATH*
If not exporting the full Path to MSYS2 (using *'-use-full-path' parameter* or *MSYS2_PATH_TYPE=inherit*) then add the path to gnupg/bin to the beginning of *PATH* by adding the following to your *.bash_profile* (or .bashrc):
```bash
PATH="/c/Program Files (x86)/gnupg/bin":$PATH
```

See below for an example [*.bash_profile*](#.bash_profile).

## **gpg** and **mintty** needs **winpty**
Some `gpg` commands, for example `gpg --edit-card` and `gpg --edit-key`, prints its output to a Windows console, which *mintty* is not. So nothing will be printed in the terminal and the program will just be stuck.

This is solved by starting `gpg` using [winpty](https://github.com/rprichard/winpty).

First install winpty using pacman:
```
$ pacman -S winpty
```

Then make an alias so that `gpg` can be started using `winpty`:
```bash
alias gpgw='winpty -- gpg'
```

Whenever the `gpg` command seems to be stuck, terminate it and try using `gpgw` instead.
The reason not to call the alias `gpg` to always use *winpty* is that some commands
does not work as expected when running it through *winpty*. So you'll need both.

Put the alias in your *.bash_profile* file, see [below](#.bash_profile).

## Enter passphrase in the terminal instead of a popup window
The default behaviour when `gpg` is asking for a password is to use a graphical popup window.

By adding `pinentry-mode loopback` to your `gpg.conf` file (`~/.gnupg/gpg.conf`) gpg will ask for passphrases using the same terminal as it was called from, but since pinentry prints its output to a Windows console then `gpg` **must** be called using `winpty`, see [above](#gpg-and-mintty-needs-winpty).

Deleting a **secret** key doesn't technically need a *pin* entry, instead a confirmation is supposed to pop up in a pinentry style, expecting 'yes' from you. If you have `pinentry-mode loopback` enabled, you'll encounter errors when deleting a secret key normally.
To solve this, use `gpgw --yes --delete-secret-keys [fingerprint]`.

Unfortunately if using the gpg-agent + ssh-pageant from below then any ssh command that accesses the key from the gpg-agent will still show the popup window.

# SmartCard SSH Authentication in MSYS2
I have a Yubikey4 which is loaded with my gpg keys that I use for SSH authentication.

To enable this I run the gpg-agent with *putty-support*, which will make the gpg-agent act as PuTTYs Pageant, and then use [*ssh-pageant*](https://github.com/cuviper/ssh-pageant) which is an ssh-agent that connects to Pageant to get the keys.

```bash
# 1. Install ssh-pageant:
$ pacman -S ssh-pageant
# 2. Add 'enable-putty-support' to gpg-agent.conf:
$ echo enable-putty-support >> $GNUPGHOME/gpg-agent.conf
# 3. (Re-)start the gpg-agent:
$ gpgconf --kill all && gpgconf --launch gpg-agent
# 4. Start the ssh-pageant:
eval $(ssh-pageant -r -a "/tmp/S.ssh-pageant.$USERNAME")
```
Run `ssh-add -L` to see the public key for the authentication key on your SmartCard.

Other programs that use Pageant, like WinSCP, can also get the key from the gpg-agent now.
The startup of the `gpg-agent` and `ssh-pageant` can be added to the [*.bash_profile*](#.bash_profile).

# .bash_profile
Add the following to *.bash_profile* to autostart the `gpg-agent` and `ssh-pageant`, unless connected via SSH, and make all the other preparations for it to work:
```bash
# Add GnuPG to PATH
PATH="/c/Program Files (x86)/gnupg/bin":$PATH
# Start gpg using winpty to get the output printed in the correct terminal
alias gpgw='winpty -- gpg'

function start_gpg_agent()
{
    # Make sure that the gpg-agent is running since ssh-pageant does not know how to start it.
    gpgconf --launch gpg-agent
    # Start ssh-pageant, use -r to reuse the "socket" if it exists.
    eval $(ssh-pageant -r -a "/tmp/S.ssh-pageant.$USERNAME") > /dev/null
}
[ -z "$SSH_CONNECTION" ] && start_gpg_agent
```

# Problems
## 'Windows HELLO for Business' and SmartCard problem with GnuPG
I had my Microsoft work account connected to my local user on my home computer, and when messing around I got a question if I wanted to enable something with that account, I don't exactly remember what.

But anyway, after enabling it my Yubikey4 stopped working with `gpg`, I got an error saying '*No card*'.

After some digging around I found that there was a new card reader installed called `Windows HELLO for Business`, and this new reader where on index 0 while my Yubikey card reader now had index 1.

I tried to remedy this by adding `reader-port 1` to *scdaemon.conf* in my *GNUPGHOME* directory but it didn't work, it seemd like the *scdaemon* didn't read the file.

I then tried to remove the new `Windows HELLO for Business` card reader but without success. The reader were not listed in the Device Manager and the internet gave no answer on how to uninstall. I began to realize that it was a big mistake to enable that thing from the beginning...

But if I logged on to another user then the new card reader were not present. So I ended up creating a new local user for myself, and this time I won't ever connect my work account.
