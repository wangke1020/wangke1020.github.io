---
title: pipe and fork
date: 2016-09-19 22:51:09
tags: linux
---

## Pipe Introduction

Every program we run on the command line automatically has three data streams connected to it.

STDIN (0) - Standard input (data fed into the program)
STDOUT (1) - Standard output (data printed by the program, defaults to the terminal)
STDERR (2) - Standard error (for error messages, also defaults to the terminal)

![streams](/img/pipe_and_fork/streams.png)

## Create a simple pipe in C

To create a simple pipe with C, we make use of the pipe() system call. It takes a single argument, which is an array of two integers, and if successful, the array will contain two new file descriptors to be used for the pipeline. After creating a pipe, the process typically spawns a new process (remember the child inherits open file descriptors).

  >SYSTEM CALL: pipe();                                                          
  >
  PROTOTYPE: int pipe( int fd[2] );                                             
    RETURNS: 0 on success                                                       
             -1 on error: errno = EMFILE (no free descriptors)                  
                                  EMFILE (system file table is full)            
                                  EFAULT (fd array is not valid)                

NOTES: fd[0] is set up for reading, fd[1] is set up for writing

code example:
```cpp
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

main()
{
    int     fd[2];
    pid_t   childpid;

    pipe(fd);
    .
    .
}
```
After pipe is invoked, fd[0] and fd[1] are connected as following picture.

![pipe](/img/pipe_and_fork/pipe.png)

<!-- more -->
### Use fork and pipe

```cpp
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

main()
{
        int     fd[2];
        pid_t   childpid;

        pipe(fd);

        if((childpid = fork()) == -1)
        {
                perror("fork");
                exit(1);
        }

        if(childpid == 0)
        {
                /* Child process closes up input side of pipe */
                close(fd[0]);
        }
        else
        {
                /* Parent process closes up output side of pipe */
                close(fd[1]);
        }
        .
        .
}
```

After fork is called, child process is created and it also have the two file descriptors.

![pipe_and_fork](/img/pipe_and_fork/pipe_and_fork.png)

If the parent wants to receive data from the child, it should close fd1, and the child should close fd0. If the parent wants to send data to the child, it should close fd0, and the child should close fd1. Since descriptors are shared between the parent and child, we should always be sure to close the end of pipe we aren't concerned with. On a technical note, the EOF will never be returned if the unnecessary ends of the pipe are not explicitly closed.

![pipe_and_fork](/img/pipe_and_fork/after_close_unused_ends.png)



 Once the pipeline has been established, the file descriptors may be treated like descriptors to normal files.

```cpp
/*****************************************************************************
 Excerpt from "Linux Programmer's Guide - Chapter 6"
 (C)opyright 1994-1995, Scott Burkett
 *****************************************************************************
 MODULE: pipe.c
 *****************************************************************************/

#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main(void)
{
        int     fd[2], nbytes;
        pid_t   childpid;
        char    string[] = "Hello, world!\n";
        char    readbuffer[80];

        pipe(fd);

        if((childpid = fork()) == -1)
        {
                perror("fork");
                exit(1);
        }

        if(childpid == 0)
        {
                /* Child process closes up input side of pipe */
                close(fd[0]);

                /* Send "string" through the output side of pipe */
                write(fd[1], string, (strlen(string)+1));
                exit(0);
        }
        else
        {
                /* Parent process closes up output side of pipe */
                close(fd[1]);

                /* Read in a string from the pipe */
                nbytes = read(fd[0], readbuffer, sizeof(readbuffer));
                printf("Received string: %s", readbuffer);
        }

        return(0);
}
```

### Reference:
* http://tldp.org/LDP/lpg/node11.html
* https://upsilon.cc/~zack/teaching/1314/progsyst/cours-03-pipe.pdf
