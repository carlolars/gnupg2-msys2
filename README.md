- [Using GnuPGv2 with MSYS2](#using-gnupgv2-with-msys2)
    - [Change GnuPG home (optional)](#change-gnupg-home-optional)
    - [Install GnuPG](#install-gnupg)
    - [Paths](#paths)
        - [Windows *Path*](#windows-path)
        - [MSYS2 *PATH*](#msys2-path)
    - [Using gpg](#using-gpg)
        - [Problems in MSYS2 terminal](#problems-in-msys2-terminal)
- [SmartCard SSH Authentication in MSYS2](#smartcard-ssh-authentication-in-msys2)
    - [Using .bash_profile to start gpg-agent and ssh-pageant](#using-bashprofile-to-start-gpg-agent-and-ssh-pageant)
- ['Windows HELLO for Business' and SmartCard problem in GnuPG](#windows-hello-for-business-and-smartcard-problem-in-gnupg)

# Using GnuPGv2 with MSYS2
MSYS2 comes with version 1.4 of GnuPG and will (probably) never get version 2.

There are mingw32/64 packets that can be installed using pacman but they don't work at all, at least not the mingw64 version I have tested.

Instead I use the official build of GnuPG from www.gnupg.org, and it (almost) works.

## Change GnuPG home (optional)
The default home directory used by GnuPG is `%APPDATA%/gnupg`.

If you want to use another path then you must create an environmental variable for your user so that both MSYS2 terminal and Windows terminal use the same home for GnuPG.

This is easiest to do from the command line using the `SETX` command, for example to set the GnuPG home to `H:\.gnupg` use the following command:
```
# Windows Terminal:
>setx GNUPGHOME H:\.gnupg
# MSYS2 Terminal:
$ setx GNUPGHOME /h/.gnupg
```

## Install GnuPG
Download and install *Gpg4win*. Gpg4win will install the same GnuPG as the *simple installer* but with some additional GUI applications and more importantly a better pinentry.

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
If not exporting the full Path to MSYS2 (using *'-use-full-path' parameter* or *MSYS2_PATH_TYPE=inherit*) then add the path to gnupg/bin to the beginning of *PATH* by adding the following to .bash_profile (or .bashrc):
```
PATH="/c/Program Files (x86)/gnupg/bin":$PATH
```
## Using gpg
Now you should be able to use `gpg` in both a Windows terminal and the MSYS2 terminal, both using the same keyrings, but unfortunately with some limitations.

### Problems in MSYS2 terminal
Some gpg commands, for example `--edit-key` and `--edit-card` does not work in the MSYS2 terminal. They just freeze without printing anyting to the console and gpg has to be terminated. (I suspect that it has something to do with the commands not printing to the correct terminal.)

Fortunately those commands work just fine in a Windows terminal, so as a workaround for now whenever a command is not working in MSYS2 terminal I switch to a Windows terminal.

# SmartCard SSH Authentication in MSYS2
I have a Yubikey4 which is loaded with my gpg keys that I use for SSH authentication.

To enable this I run the gpg-agent with *putty-support*, which will make the gpg-agent act as a PuTTY Pageant, and then use *ssh-pageant* (https://github.com/cuviper/ssh-pageant) which is an ssh-agent that connects to a Pageant to get the keys.

1. Install *ssh-pageant*: `$ pacman -S ssh-pageant`
2. Add '*enable-putty-support*' to gpg-agent.conf: `$ echo enable-putty-support >> $GNUPGHOME/gpg-agent.conf`
3. (Re-)start the gpg-agent: `gpgconf --kill all && gpgconf --launch gpg-agent`
4. Start the ssh-pageant: `eval $(ssh-pageant -r -a "/tmp/S.ssh-pageant.$USERNAME")`
5. Run `ssh-add -L` to see the public key for your SmartCard.

Other programs that use Pageant, like WinSCP, can also get the key from the gpg-agent now.

## Using .bash_profile to start gpg-agent and ssh-pageant
Add the following to *.bash_profile* to autostart the `gpg-agent` and `ssh-pageant` unless connected via SSH:
```
function setup_gpg2()
{
  # Add GnuPG to PATH
  PATH="/c/Program Files (x86)/gnupg/bin":$PATH
  # Make sure that the gpg-agent is running since ssh-pageant does not know how to start it.
  gpgconf --launch gpg-agent
  # Start ssh-pageant, use -r to reuse the "socket" if it exists.
  eval $(ssh-pageant -r -a "/tmp/S.ssh-pageant.$USERNAME") > /dev/null
}
[ -z "$SSH_CONNECTION" ] && setup_gpg2
```
# 'Windows HELLO for Business' and SmartCard problem in GnuPG
I had my Microsoft work account connected to my local user on my home computer, and when messing around I got a question if I wanted to enable something with that account, I don't exactly remember what.

But anyway, after enabling it my Yubikey4 stopped working with `gpg`, I got an error saying '*No card*'.

After some digging around I found that there was a new card reader installed called `Windows HELLO for Business`, and this new reader where on index 0 while my Yubikey card reader now had index 1.

I tried to remedy this by adding `reader-port 1` to *scdaemon.conf* in my *GNUPGHOME* directory but it didn't work, it seemd like the *scdaemon* didn't read the file.

I then tried to remove the new `Windows HELLO for Business` card reader but without success. The reader were not listed in the Device Manager and the internet gave no answer on how to uninstall. I began to realize that it was a big mistake to enable that thing from the beginning...

But if I logged on to another user then the new card reader were not present. So I ended up creating a new local user for myself, and this time I won't ever connect my work account.
