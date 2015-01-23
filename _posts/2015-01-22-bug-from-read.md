---
layout: post
published: false
title: RTFM: A bug caused by read
excerpt:
tags: lxc pipe read linux core
---
{% include JB/setup %}

This is a bug coming up in work. It's very interesting and it's a good example of why we need to stick to good coding style. Also, always remember, when issue comes up, remember to RTFM.

Here is the related code snippet (simplified version):

```
// fd is opened correctly
// buffer has enough space and it has been stuffed with 0s
int read_from_fd (FILE* fd, char* buffer, size_t buf_size) {
    int nd = 0;
    nd = read(fd, buffer, buf_size);
    if (nd < 0) {
        // error handling
    }
    // process buffer
}
```

What could go wrong?

The issue is this: nd is not necessarily equal to buf_size, especially when reading from pipe. If you are using a slow simulated environment and sending big chuck of mem through pipe between lxc, chances are that you would hit this issue.

Return Value of read() from [man](http://linux.die.net/man/2/read) page:

> On success, the number of bytes read is returned (zero indicates end of file), and the file position is advanced by this number. It is not an error if this number is smaller than the number of bytes requested; this may happen for example because fewer bytes are actually available right now **(maybe because we were close to end-of-file, or because we are reading from a pipe, or from a terminal)**, or because read() was interrupted by a signal. On error, -1 is returned, and errno is set appropriately. In this case it is left unspecified whether the file position (if any) changes.

So, (nd < 0) is not enough. It is advisable to check (nd == buf_size) too.

Here is a fixed version:

```
// fd is opened correctly
// buffer has enough space and it has been stuffed with 0s
int read_from_fd (FILE* fd, char* buffer, size_t buf_size) {
    int nd = 0;
    while (nd != buf_size)
        nd += read(fd, buffer + nd, buf_size - nd);
        if (nd < 0) {
            // error handling
        }
    }
    // process buffer
}
```

