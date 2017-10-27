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
