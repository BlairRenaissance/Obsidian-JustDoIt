
## What is the shell

**textual interface**: The Shell.

Nearly all platforms you can get your hand on has a shell in one form or another, and many of them have several shells for you to choose from. While they may vary in the details, at their core they are all roughly the same: they allow you to run programs, give them input, and inspect their output in a semi-structured way.

## Using the shell

This is the main textual interface to the shell. 
```bash
missing:~$
```
It tells you that you are on the machine `missing` and that your “current working directory”, or where you currently are, is `~` (short for “home”). The `$` tells you that you are not the root user.

```
missing:~$ ls -l /home
drwxr-xr-x 1 missing  users  4096 Jun 15  2019 missing
```
First, the `d` at the beginning of the line tells us that `missing` is a directory. Then follow three groups of three characters (`rwx`). These indicate what permissions the owner of the file (`missing`), the owning group (`users`), and everyone else respectively have on the relevant item. A `-` indicates that the given principal does not have the given permission. Above, only the owner is allowed to modify (`w`) the `missing` directory (i.e., add/remove files in it). To enter a directory, a user must have “search” (represented by “execute”: `x`) permissions on that directory (and its parents). To list its contents, a user must have read (`r`) permissions on that directory. For files, the permissions are as you would expect. Notice that nearly all the files in `/bin` have the `x` permission set for the last group, “everyone else”, so that anyone can execute those programs.