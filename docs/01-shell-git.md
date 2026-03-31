---
title: Linux Shell and Git
---

# Lab Assignment 0 Linux Shell and Git

The following is a crash course on using the command line on UNIX or GNU/Linux systems. The prompt in these examples is simply written `$`; on the server, it will look like:

```
[USER@cdmcscaoprd01 ~]$
```

## 1. Connecting and transferring files
### 1.1. SSH client

You will connect to the server `matrix.cdm.depaul.edu` (**cdmcscaoprd01**) via [`SSH`](https://wikipedia.org/wiki/SSH) and clone the lab files using [`Git`](https://git-scm.com/book/en/v2). If you use Mac OS or Linux, these are standard programs accessible through any terminal. If you use Windows, then I recommend using [PowerShell](https://learn.microsoft.com/en-us/powershell/scripting/learn/ps101/01-getting-started?view=powershell-7.5), or [WSL 2](https://learn.microsoft.com/en-us/windows/wsl/install), or [Git BASH](https://gitforwindows.org/), or [PuTTy](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html).

### 1.2. Logging in
Use your lowercase DePaul Campus Connect account name and password to log into the server (e.g., ALEX42 becomes alex42). When connecting the matrix machine, you will need to use the universitry dedicated bastion/jump host:

```
$ ssh -J alex42@sshjump.depaul.edu alex42@matrix.cdm.depaul.edu
  ...
Please enter your DePaul BlueKey password.
(alex42@sshjump.depaul.edu) Entra Id Password: CAMPUS CONNECT PASSWORD
Open your Authenticator app, and enter the number '42' to sign in.
No push? Check your mobile device's internet connection.
  ...
alex42@matrix.cdm.depaul.edu's password: CAMPUS CONNECT PASSWORD
[alex42@cdmcscaoprd01 ~]$
```

More information about it can be found here:  
[https://depaul.service-now.com/sp/en/ssh-proxy-jump-host?id=kb_article_view&sysparm_article=KB0013112](https://depaul.service-now.com/sp/en/ssh-proxy-jump-host?id=kb_article_view&sysparm_article=KB0013112)

**Note:** If you had to use that server in a previous session, you may get an error message along the lines of:

```
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
...
Offending ECDSA key in PATH/.ssh/known_hosts:N
```

If that happens, you can safely ignore this message; to do so, delete line `N` in the file `PATH/.ssh/known_hosts` (changing the values of `N` and `PATH` depending on the above error message), then reconnect.

### 1.3. Logging in with an SSH-agent

I recommend that you pick a randomly generated password and then you create an SSH key to log in conveniently. The idea of SSH keys is that:
- You generate a private/public key pair on your home computer (using [`ssh-keygen(1)`](https://man7.org/linux/man-pages/man1/ssh-keygen.1.html) and automatically stored in `~/.ssh/`),
- You put the public key on the remote computer (using `ssh-copy-id USER@SERVER` or manually by adding it to the remote file `~/.ssh/authorized_keys`).
- Subsequent logins from your home computer will be asking for the passphrase of your key (if any) rather than your password and you can use [`ssh-agent(1)`](https://man7.org/linux/man-pages/man1/ssh-agent.1.html) to have to type your password only once per session.

[This page](https://wiki.archlinux.org/index.php/SSH_keys) has everything you didn’t know you wanted to know about SSH keys, while [this one](https://phoenixnap.com/kb/generate-ssh-key-windows-10) covers how to generate keys on Windows 10 (Mac domain users can follow the Linux way).

### 1.4. Step-by-step commands for setting up public-private key authentication

Generate a new public-private key pair using default location:

```shell
local-machine$ ssh-keygen
```

Grab the content of the public key:

```shell
local-machine$ cat .ssh/id_ed25519.pub
YOUR-PUBLIC-KEY
```

Copy the public key to the bastion/jump host by accessing [https://keymgmt.sshjump.depaul.edu/](https://keymgmt.sshjump.depaul.edu/).

Log into the remote machine, create and copy the public key to authorized_keys:

```shell
local-machine$ ssh -J alex42@sshjump.depaul.edu USER@matrix.cdm.depaul.edu
remote-machine$ mkdir .ssh # create the .ssh directory if it does not exist
remote-machine$ touch .ssh/authorized_keys # create the authorized_keys file if it does not exists
remote-machine$ echo "YOUR-PUBLIC-KEY" >> .ssh/authorized_keys # append your public key to authorized_keys
remote-machine$ chmod 755 .ssh; chmod 644 .ssh/authorized_keys # set the proper permissions

```

Now you should be able to to log in to the remote machine using the public-private key pair instead of using the password.

### 1.5. About paths in Linux

A path in UNIX/Linux is a string of the shape `/usr/bin/ls`; this refers to the file [`ls(1)`](https://man7.org/linux/man-pages/man1/ls.1.html) in the subdirectory `bin` of the directory `usr` that can be found at the root of the filesystem. In there, the first `/` indicates that the path is absolute, that is, it will point to the same file wherever it is mentioned. The relative path `bin/ls` refers to the file [`ls(1)`](https://man7.org/linux/man-pages/man1/ls.1.html) in the subdirectory `bin` of the current directory.

To retrieve the current directory, run:

```shell
$ pwd
/home/USER
```

To change path, run:

```shell
$ cd /home
$ pwd
/home
```

There are a handful of special paths:
- `.` is the current directory.
- `..` is the parent directory.
- `~` is your home.
- `~USER` is the home of `USER`.

```shell
$ cd ~/././.././
$ pwd
/
```

### 1.6. Transferring files

Suppose you have a file `test.txt` on your local computer, say on your desktop. To transfer it to the server using `scp`, use:

```shell
$ scp Desktop/test.txt USER@matrix.cdm.depaul.edu:PATH
```

The text `PATH` can be any path (absolute or relative). If it is an existing directory, it will put the `test.txt` in it. If it is a file, it will overwrite the file with `test.txt`. If no path is specified, `test.txt` is put in your home.

To retrieve `PATH/test.txt` from the server, simply invert the two arguments:

```shell
$ scp USER@matrix.cdm.depaul.edu:PATH/test.txt Desktop/test.txt
```

## 2. Programs, binaries, executables, commands

I will use these terms interchangeably. They refer to files that can (or more precisely, can and are meant to) be executed.

### 2.1. Running programs on the command line

Consider the following situation:

```shell
$ scp -x test.txt
unknown option -- x
usage: scp [-12346BCpqrv] [-c cipher] [-F ssh_config] [-i identity_file]
           [-l limit] [-o ssh_option] [-P port] [-S program]
           [[user@]host1:]file1 ... [[user@]host2:]file2
```

What happens is the following:
- The shell reads your input ([`scp(1)`](https://man7.org/linux/man-pages/man1/scp.1.html)).
- The shell looks in agreed-upon places on the filesystem to find a program bearing that name (the list of such paths is stored in the environment variable `PATH`, the value of which can be found by invoking [`env(1)`](https://man7.org/linux/man-pages/man1/env.1.html) on the command line).
- The shell finds that `/usr/bin/scp` is an existing executable file, so it runs it, giving as argument to the program the strings `"-x"` and `"test.txt"`.
- The program [`scp(1)`](https://man7.org/linux/man-pages/man1/scp.1.html) then outputs that it did not understand the arguments, and indicates correct usage. It tells us that [`scp(1)`](https://man7.org/linux/man-pages/man1/scp.1.html) has a number of possible switches (starting with `-`), and these are optional (this is the meaning of the squared brackets). It also shows that [`scp(1)`](https://man7.org/linux/man-pages/man1/scp.1.html) has a number of normal arguments (`file1`, `...`, `file2`); these are compulsory.

To know which program is actually executed, one can do:

```shell
$ which scp
/usr/bin/scp
```

Note that the current path `.` should never be in the `PATH` variable (this is a security breach). This means that if you compiled a file to the program `hello` in the current directory, you should use `./hello` to run it.

### 2.2. Documentation on programs

When you want documentation on a program, use the command [`man(1)`](https://man7.org/linux/man-pages/man1/man.1.html) followed by the name of the program:

```shell
$ man scp
SCP(1)                   BSD General Commands Manual                   SCP(1)

NAME
     scp — secure copy (remote file copy program)

SYNOPSIS
     scp [-346BCpqrTv] [-c cipher] [-F ssh_config] [-i identity_file]
         [-J destination] [-l limit] [-o ssh_option] [-P port] [-S program]
         source ... target
  ...
```

Use the keyboard arrows to navigate the page, and `q` to quit.

It may be that the program has a lengthier documentation, or a documentation put in the context of other programs. This would be provided by [`info(1)`](https://man7.org/linux/man-pages/man1/info.1.html):

```shell
$ info cat
Next: tac invocation,  Up: Output of entire files

3.1 ‘cat’: Concatenate and write files
======================================

‘cat’ copies each FILE (‘-’ means standard input), or standard input if
none are given, to standard output.  Synopsis:

     cat [OPTION] [FILE]...

   The program accepts the following options.
  ...
```

Arrows are used to move around, and `q` to quit.

## 3. Cloning the repository using Git

[`Git`](https://git-scm.com/) is a distributed version control system that tracks changes in source code during software development. It is the industry standard for version control. `Git` usually operates on a repository (often shortened to `repo`), which is a storage location where your project's files and revision history are kept. A project may have multiple `repos`, each of them potentially located on a differen machine.

### 3.1. Basic Git Commands

`Git` provides you with a set of primitives that allows you to synchronize your repository with a source repository. Here are some of the most common `Git` commands you will use:

- `git status`: Displays the state of the working directory and the staging area of your repo.
- `git add <file>`: Adds a change in the working directory to the staging area of your repo.
- `git commit -m "message"`: Captures a snapshot of the project's currently staged changes.
- `git push`: Uploads your local repository content to a remote repo.
- `git pull`: Fetches and downloads content from a remote repos and immediately updates the local repo to match that content.

### 3.2 Cloning the Lab Assignment 0 Repo

To start the lab, you must clone the starter code repository. Run the following command:

```shell
$ cd ~
$ git clone https://github.com/transcendental-software/csc-397-595-lab0.git
```

This command creates a directory containing all the necessary files for the assignment. You can then navigate into it using `cd`.

## 4. Manipulating files

### 4.1. List files

The command [`ls(1)`](https://man7.org/linux/man-pages/man1/ls.1.html) is used to list files. As an argument, it can take the files to be listed, or a directory:

```shell
$ cd ~/csc-397-595-lab0/bos
$ ls
dir1  dir2  dir3  f1  f2  f3
$ ls ~/csc-397-595-lab0/bos
dir1  dir2  dir3  f1  f2  f3
```

To get more information on the files, use the option `-l`:

```shell
$ ls -l ~/csc-397-595-lab0/bos
total 0
drwx------ 2 USER domain users 6 Jan  7 17:51 dir1
drwxr-xr-x 2 USER domain users 6 Jan  7 17:51 dir2
drwx--x--x 2 USER domain users 6 Jan  7 17:51 dir3
-rw------- 1 USER domain users 0 Jan  7 17:51 f1
-rwxr-xr-x 1 USER domain users 0 Jan  7 17:51 f2
-rw-r--r-- 1 USER domain users 0 Jan  7 17:51 f3
```

The first column indicates the rights on the file. It is to be read as:
- First character: type of file (for our purposes, directory or regular file)
- Next three blocks of three characters are the rights for the owner of the file, the group of the owner, and others, respectively:
  - The first letter is `r` if the file can be read (for regular files) or the directory can be listed (for directories).
  - The second letter is [`w(1)`](https://man7.org/linux/man-pages/man1/w.1.html) if the file can be modified (for regular files) or the directory content can be changed (for directories).
  - The third letter is `x` if the file can be executed (for regular files) or the directory can be traversed (for directories).

You wouldn’t be able to go into `dir1`, read the file `f1`,

A hidden file is a file whose name starts with a period. We have already seen two “special” hidden files: `.`, the current directory, and `..`, the parent directory. To view hidden files when listing, use `-a`:

```shell
$ ls -l -a ~/csc-397-595-lab0/bos
total 0
drwxrwxr-x 5 USER domain users 91 Jan  7 17:53 .
drwxr-xr-x 4 USER domain users 38 Jan  7 17:51 ..
drwx------ 2 USER domain users  6 Jan  7 17:51 dir1
drwxr-xr-x 2 USER domain users  6 Jan  7 17:51 dir2
drwx--x--x 2 USER domain users  6 Jan  7 17:51 dir3
-rw------- 1 USER domain users  0 Jan  7 17:51 f1
-rwxr-xr-x 1 USER domain users  0 Jan  7 17:51 f2
-rw-r--r-- 1 USER domain users  0 Jan  7 17:51 f3
-rw-rw-r-- 1 USER domain users  0 Jan  7 17:53 .hiddenfile
```

### 4.2. Copying, moving (renaming), removing files.

To copy files, use [`cp(1)`](https://man7.org/linux/man-pages/man1/cp.1.html). To move files (and rename them) use [`mv(1)`](https://man7.org/linux/man-pages/man1/mv.1.html). To delete files use [`rm(1)`](https://man7.org/linux/man-pages/man1/rm.1.html).

```shell
$ cd ~
$ cp ~/csc-397-595-lab0/bos/f2 .
$ ls
csc397-595-lab0 f2
$ mv f2 g2
$ ls
g2
$ rm g2
$ ls
```

To copy and remove directories, use the option `-r`:

```shell
$ cp ~/csc-397-595-lab0/bos .
cp: omitting directory ‘/home/USER/csc-397-595-lab0/bos’
$ cp -r ~/csc-397-595-lab0/bos lecture0
$ ls
csc-397-595-lab0 lecture0
$ ls lecture0
dir1  dir2  dir3  f1 f2  f3
$ rm -r lecture0
$ ls lecture0
ls: cannot access lecture0: No such file or directory
```

Lots of commands take `-r` or `-R` as an option to go recursively, that is, through subdirectories (e.g., `ls -R`).

### 4.3. Creating empty files and directories

To create empty files, use [`touch(1)`](https://man7.org/linux/man-pages/man1/touch.1.html), to create an empty directory, use [`mkdir(1)`](https://man7.org/linux/man-pages/man1/mkdir.1.html):

```shell
$ mkdir dir1 dir2
$ touch dir1/a dir2/b
$ ls -R -l
.:
total 0
drwxrwxr-x 2 USER domain users 15 Jan  7 18:07 dir1
drwxrwxr-x 2 USER domain users 15 Jan  7 18:07 dir2

./dir1:
total 0
-rw-rw-r-- 1 USER domain users 0 Jan  7 18:07 a

./dir2:
total 0
-rw-rw-r-- 1 USER domain users 0 Jan  7 18:07 b
$ rm -r dir1 dir2
$ ls
```

## 5. Viewing and editing files

### 5.1. Viewing files

The main commands to view files quickly are [`cat(1)`](https://man7.org/linux/man-pages/man1/cat.1.html) and [`less(1)`](https://man7.org/linux/man-pages/man1/less.1.html). [`cat(1)`](https://man7.org/linux/man-pages/man1/cat.1.html) reads the files provided as arguments, and prints them verbatim:

```shell
$ cp ~/csc-397-595-lab0/boc/hello.c .
$ cat hello.c
#include <stdio.h>

int main()
{
  printf("hello, world\n");
  return 42;
}
$ cat hello.c hello.c
#include <stdio.h>

int main()
{
  printf("hello, world\n");
  return 42;
}
#include <stdio.h>

int main()
{
  printf("hello, world\n");
  return 42;
}
```

[`less(1)`](https://man7.org/linux/man-pages/man1/less.1.html) is more interactive, and allows you to navigate through the file. Within [`less(1)`](https://man7.org/linux/man-pages/man1/less.1.html), I mostly use the following commands:
- `q`: exit.
- `/`: search for a string (then `n` to get to the next match).
- `-i`: toggle case sensitivity in search.

### 5.2. Editing files

[ed is the standard text editor](https://www.gnu.org/fun/jokes/ed-msg.txt), but no one uses it anymore. I recommend you become familiar with [`vim`](https://www.vim.org/) or [`emacs(1)`](https://man7.org/linux/man-pages/man1/emacs.1.html).

If you simply run `vim FILE`, it will open `FILE` in normal mode. Press `i` to enter insert mode and start editing. To save, press `Esc` to return to normal mode, then type `:w` and press Enter. To quit, type `:q`. To save and quit in one step, use `:wq` (here, Vim notation uses `:` to enter command-line mode, and keys like `Esc` to switch modes).

[`vim`](https://www.vim.org/) is a highly efficient modal text editor, famous for its steep learning curve and exceptional editing speed once mastered. I strongly encourage you to run `vimtutor`, which is included with `vim` and takes about 30 minutes to complete. It provides a hands-on introduction to the core concepts—modes, movement, editing, and commands. Time spent learning `vim` fundamentals pays off quickly and compounds over years of use.

If you simply run `emacs FILE`, it will open `FILE`, you can then save with `C-x C-s` and exit with `C-x C-c` (here, the Emacsian notation `C-KEY` is used to indicate `KEY` pressed together with the `CTRL` control key).

[`emacs(1)`](https://man7.org/linux/man-pages/man1/emacs.1.html) is a powerful work environment, sometimes mocked as being a full OS. I strongly encourage you to read [Jess Hamrick’s Emacs Tutorial](https://www.jesshamrick.com/post/2012-09-10-absolute-beginners-guide-to-emacs/). It is superbly easy to follow, and will get you introduced to the very basics of Emacs in less than 30 minutes. This is time you will not regret investing.