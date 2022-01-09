### Lab8 file system

#### Large files

Implement "doubly-indirect" block in the inode, to make each inode be able to contain `256*256+256+11` blocks. Recall the definition of inode and how `direct and indirect block` works. This lab is much easy to accomplish.



Just follow the *hints* and modify the `dinode` and `inode` for the new `double-indirect` block. 



#### Symbolic links

The following sentence refers to manual page in linux of symlink.

##### man symlink

> symlink() creates a symbolic link named linkpath which contains the string target.
>
>  Symbolic links are interpreted at run time as if the contents of the link had been substituted into the path being followed to find a file or directory.
>
>  Symbolic links may contain ..  path components, which (if used at the start of the link) refer to the parent directories of that in which the link resides.
>
>  A symbolic link (also known as a soft link) may point to an existing file or to a nonexistent one; the latter case is known as a dangling link.
>
>  The permissions of a symbolic link are irrelevant; the ownership is ignored when following the link, but is checked when removal or renaming of the link is requested and the link is in a directory with the sticky bit (S_ISVTX) set.
>
> If linkpath exists, it will not be overwritten.

##### man open (about O_NOFOLLOW)

> If pathname is a symbolic link, then the open fails, with the error ELOOP.  Symbolic links in earlier components of the pathname will still be followed.  (Note  that  the **ELOOP** error that can occur in this case is indistinguishable from the case where an open fails because there are too many symbolic links found while resolving components in the prefix part of the pathname.)

So if **open()** calls for some file with `O_NOFOLLOW` just ignore the symbolic link flag and directly return the block. If without `O_NOFOLLOW` , we need do search recursively till find the first non-symlink inode or exceed 10-depth limit.



In order to do so, firstly implement `sys_symlink()`:

I put  the symlink information into the first block of the inode.

```c
uint64 sys_symlink(void){
  char target[MAXPATH], path[MAXPATH];
  struct inode *ip;
  if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0){
    return -1;
  }
  begin_op();
  //To create a symlink in path
  ip = create(path, T_SYMLINK, 0, 0);
  if(ip == 0){
    end_op();
    return -1;
  }
  if(writei(ip, 0, (uint64)target, 0, MAXPATH) < 0){
    end_op();
    iunlockput(ip);
    return -1;
  }

  iunlockput(ip);

  end_op();
  return 0;
}
```

Then to open a path which may be a symbolic link on it:

```c
  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    if((ip = namei(path)) == 0){
      end_op();
      return -1;
    }
    ilock(ip);
    while(ip->type == T_SYMLINK){
      if((omode & O_NOFOLLOW) == 0){
        //follow sym link
        if(++depth > 10){
          iunlockput(ip);
          end_op();
          return -1;
        }
        if(readi(ip, 0, (uint64)path, 0, MAXPATH) < 0){
          iunlockput(ip);
          end_op();
          return -1;
        }
        iunlockput(ip);
        if((ip = namei(path)) == 0){
          end_op();
          return -1;
        } 
        ilock(ip);
      }else{
        break;
      }
    }
    ...
    }
```

