---
title: Function Structures and Cleanup.
layout: default
---

# Function Structures and Cleanup.

With socket programming, there are often functions that have a sequence of steps
allocating or binding resources. These call for a function structure where any
failure step clean up all resources allocated up to that point. There's a few
ways to handle this, but I'll start with some of my least favourite ones. The
example will be a client connecting, to which a heap-allocated memory structure
is reserved. So there's an accept-call for the fd, an allocation for the memory
area and a (man 7 epoll) epoll_ctl call for binding the fd to the main programs
event loop.

## Making a list and tearing it down twice.

First the repetitive way of tearing down everything brought up to that point,
each in its own clause.

```c
struct client_context *client_accept(int main_fd, int epoll_fd)
{
  struct client_context *client_ctx;
  struct epoll_event ev = { };
  int fd = accept(main_fd, NULL, NULL);

  if (fd == -1)
    {
      perror("accept");
      return NULL;
    }

  client_ctx = calloc(1, sizeof(*client_ctx));
  if (!client_ctx)
    {
      perror("calloc");
      close(fd);
      return NULL;
    }
  client_ctx->fd = fd;

  ev.data.fd = fd;
  ev.events = EPOLLIN;
  if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &ev))
    {
      perror("epoll_ctl");
      free(client_ctx);
      close(fd);
      return NULL;
    }
  // Success.
  return client_ctx;
}
```

Now a few obvious annoyances here: The closing of the fd has to be repeated on
any subsequent step which can fail. This can be avoided by using a goto in the
end.

```c
struct client_context *client_accept_goto(int main_fd, int epoll_fd)
{
  struct client_context *client_ctx = NULL;
  struct epoll_event ev = { };
  int fd = accept(main_fd, NULL, NULL);

  if (fd == -1)
    {
      perror("accept");
      return NULL;
    }

  client_ctx = calloc(1, sizeof(*client_ctx));
  if (!client_ctx)
    {
      perror("calloc");
      goto fail;
    }
  client_ctx->fd = fd;

  ev.data.fd = fd;
  ev.events = EPOLLIN;
  if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &ev))
    {
      perror("epoll_ctl");
      goto fail;
    }
  // Success.
  return client_ctx;

fail:
  free(client_ctx);
  close(fd);
  return NULL;
}
```

Now there's only one repetition of the tear down of failure. But not the
client_ctx now needs to be initialized to null to avoid freeing an uninitialized
pointer. This gets more complicated with fds as they should be initialized to -1
if they are set somewhere other than in the first step. In the above example
someone might add another allocating step before the fd is accepted, in which
case the goto would be missing and resources would be leaked.

Both of the these examples also have all stack-related variables on the start of
the function. Now this might be a matter of taste, but I'd prefer myself to keep
declarations of stack variables as close to the action as possible. This of
course depends on the coding style, if they're allowed to be declared other than
in the start of the block.

## Success Means a New Block

Over the years my personal favourite way has been solidified to opening a new
block on success, handling the error logging in the else and tearing down at the
end of a block.

```c
struct client_context *client_accept_blocks(int main_fd, int epoll_fd)
{
  int fd = accept(main_fd, NULL, NULL);

  if (fd != -1)
    {
      struct client_context *client_ctx = calloc(1, sizeof(*client_ctx));

      if (client_ctx)
        {
          struct epoll_event ev =
            {
              .events = EPOLLIN,
              .data =
                {
                  .fd = fd,
                }
            };
          client_ctx->fd = fd;

          if (!epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &ev))
            {
              // Success.
              return client_ctx;
            }
          else
            {
              perror("epoll_ctl");
            }
          free(client_ctx);
        }
      else
        {
          perror("calloc");
        }
      close(fd);
    }
  else
    {
      perror("accept");
    }
  return NULL;
}
```

Here all tear downs are written exactly once. Especially when the clause is
being written, the free/close can be written in to the block immediately so it's
not forgotten. Also the stack variable being used is declared and usually set at
the start of the block, in which it is closest to the code using the variable.

The flow is a bit different from the traditional though. The success steps are
the first to be done, meaning the most relevant information on the function is
actually at the top. And all that error logging, always dropped for brevity, is
left to the end of the function.

This does of course limit the number of steps in the function, which might be a
good thing. Any number of non-allocating steps can be combined to a single
if-clause, e.g.

```c
  if (!non_allocating_thing_zero_on_success(...) &&
      !non_allocating_thing_zero_on_success_2(...) &&
      !non_allocating_thing_zero_on_success_3(...))
    {
      ...
```

Only of course if you can get the error logging to stay accurate. This is
usually the case in higher level parts, where the sub functions have their own
error logging pinpointing the failure clause.

## Steps Without Allocations

There's an extra pattern I learned from a colleague during code review.
If none of the steps have allocations, the need for tear down is removed. The
thing that bugs me in these situations is usually all of the multiple
return-calls when each failure has a dedicated error call. Lets take an example
from a series of fcntl operations

```c
/** Sets given \p fd as non-blocking and closing on exec. */
bool fd_operations(int fd)
{
  int status;

  if ((status = fcntl(fd, F_GETFL, NULL, 0)) == -1)
    {
      perror("F_GETFL");
      return false;
    }

  status |= O_NONBLOCK;
  if (fcntl(fd, F_SETFL, status))
    {
      perror("F_SETFL");
      return false;
    }

  if ((status = fcntl(fd, F_GETFD, NULL, 0)) == -1)
    {
      perror("F_GETFD");
      return false;
    }

  status |= FD_CLOEXEC;
  if (fcntl(fd, F_SETFD, status))
    {
      perror("F_SETFD");
      return false;
    }
  return true;
}
```

Creating an if-else chain reduces the number of exits for the function to two
and drops all extra "return false"-statements:

```c
/** Sets given \p fd as non-blocking and closing on exec. */
bool fd_operations(int fd)
{
  int status;

  if ((status = fcntl(fd, F_GETFL, NULL, 0)) == -1)
    {
      perror("F_GETFL");
    }
  else if (fcntl(fd, F_SETFL, status | O_NONBLOCK))
    {
      perror("F_SETFL");
    }
  else if ((status = fcntl(fd, F_GETFD, NULL, 0)) == -1)
    {
      perror("F_GETFD");
    }
  else if (fcntl(fd, F_SETFD, status | FD_CLOEXEC))
    {
      perror("F_SETFD");
    }
  else
    {
      return true;
    }
  return false;
}
```

In summary, none of these are the silver bullet for not leaving memory or
resource leaks in code, but some of these structures have reduced a lot of these
errors in my own code and reduced code duplication.
