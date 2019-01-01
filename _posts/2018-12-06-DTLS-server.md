---
title: DTLS Server Handling, Journey To UDP Server Land.
layout: default
---

# DTLS Server Handling, Journey to UDP Server Land.

When I first ended up tinkering with DTLS and UDP sockets in a server manner,
the question arrived fast: How is this even supposed to work. You create a
UDP socket, bind it to an port and maybe an IP. Normally each message would
be fetched with recvmsg, which allows having the peer sending the message at
hand.

```c
data(int fd)
{
  ssize_t ret;

  do
    {
      struct sockaddr_storage saddr;
      socklen_t slen = sizeof(saddr);
      char buf[BUFSIZ];

      // Read and store given peer to saddr.
      ret = recvfrom(fd, buf, sizeof(buf), 0, (struct sockaddr *)&saddr,
                     &slen);
      if (ret > 0)
        {
          // Process client data, replying to peer.
          process(fd, buf, slen, &saddr, slen);
        }
    } while (ret > 0);
}
```

But for DTLS the context would need to remain longer than just when processing
single messages from peers. Even the handshake takes multiple steps, including
making sure the client owns its source IP/port by requesting a reply with a
random cookie. If you just spin up an UDP fd (file descriptor) and call:

```c
  struct sockaddr_storage dtls_peer = {};
  int ret = DTLSv1_listen(client->ssl, &dtls_peer);
```
This ends up with ret being -1 and SSL_get_error returning SSL_ERROR_WANT_READ.
Sure you could eventually continue the connection with another call, but how
would you separate the connections coming in in-between this and whenever the
client bothers to reply? A blocking socket is never an option. It always ends up
with somebody creating a thread, or worse forks a process, to handle the
blocking situation. That's a topic for another rant though.

## A Connected and Bound Socket is More Precise than just a Bound Socket.

After a while I figured that a socket UDP socket that is only bound (man 2 bind)
will receive data from any peer sending it to the IP/port the server is bound
to. But a UDP socket that's bound and connected (man 2 connect) to a peer will
receive only data from the connected peer (obviously), but in addition the
unconnected socket will not see this data. It works similarly to routing: If
there's a more precise route, the more precise route will be used, although the
traffic would match the more specific one. Similarly a bound and connected
socket will get all the data from a given peer. So using the peer address from
before we could create a new socket:

```c
int new_peer(int main_fd, struct sockaddr_storage *peer, socklen_t peer_len)
{
  struct sockaddr_storage bound_to;
  socklen_t bound_to_len = sizeof(bound_to);

  // Query where main_fd is bound to.
  if (!getsockname(main_fd, (struct sockaddr *)&bound_to, &bound_to_len))
    {
      int yes = 1;
      int client_fd;

      // Create a new UDP socket.
      if ((client_fd =
             socket(peer->ss_family, SOCK_DGRAM | SOCK_NONBLOCK, 0)) != -1)
        {
          // Make sure the UDP socket can use the same end-point as the main fd.
          if (!setsockopt(client_fd, SOL_SOCKET, SO_REUSEADDR, &yes,
                          sizeof(yes)))
            {
              // Bind the server side.
              if (!bind(client_fd, (struct sockaddr *)&bound_to, bound_to_len))
                {
                  // Connect the peer side.
                  if (!connect(client_fd, (struct sockaddr *)peer, peer_len))
                    {
                      // Success.
                      return client_fd; // New more "precise" fd created!
                    }
                }
            }
          close(client_fd);
        }
    }
  return -1;
}
```

## The Spell of REUSEADDR and REUSEPORT

Creating new sockets bound to the same end-points requires using SO_REUSEADDR
and SO_REUSEPORT. For this UDP server scheme the main fd is set to SO_REUSEADDR
and SO_REUSEPORT, but importantly the more precise connected sockets must have
only SO_REUSEADDR. man 7 socket hints that SO_REUSEPORT would cause the new
socket to help load-balance the original socket, which isn't what we need when
we want to separate the clients.

## Hello is Already in the Main FD.

But uhm the data is already in the main fd when you receive a new peer? My lazy
first approach was to just create the socket and let the second try from the
client arrive to the more precise socket. But it just doesn't feel like the
right solution to discard any clients first connection.

In order not to waste the first Hello the SSL context being accepted with
DTLSv1_listen has the main fd as its BIO dgram. That way the first hello is
properly replied, after which the main fd is detached from the newly created SSL
context dgram BIO. But on non-blocking socket, DTLSv1_accept might not return
the new connected peer address, so how can we divert this UDP traffic to the new
socket? And what if the client replies before we the server has time to create
a dedicated fd for the socket?

## Peek don't Tell.

What if the socket could already be there before we accept with DTLS? This is
possible by using the MSG_PEEK flag for the recvmsg family. This way we can
create a dedicated, connected, specific socket for the client before the client
has time to be too fast for the server side. Then after the first DTLSv1_listen
we start using the new fd for this SSL context and continue smooth sailing the
rest of the handshake. The plan boils down to the steps:

1. Peek the peer sending the hello message.
2. Solve or use a stored address for the server end-point.
3. Create a new socket bound to server end-point, connected to peer. This will
   now receive any subsequent messages from this peer.
4. Temporarily use the main_fd in the client SSL to reply to this clients Hello.
5. Replace the client SSL fd with the newly created fd.

```c
int new_peer(int main_fd, SSL *ssl)
{
  char buf[BUFSIZ];
  struct sockaddr_storage peer, bound_to;
  socklen_t peer_len = sizeof(peer), bound_to_len = sizeof(bound_to);
  int client_fd;

  // Receive any data from peer. Just peeked, so data is irrelevant.
  if (recvfrom(main_fd, buf, sizeof(buf), MSG_PEEK, (struct sockaddr *)&peer,
               &peer_len) > 0 &&

      // Solve server end-point
      !getsockname(main_fd, (struct sockaddr *)&bound_to, &bound_to_len) &&

      // Create a new DGRAM socket as the client socket.
      (client_fd = socket(peer.ss_family, SOCK_DGRAM | SOCK_NONBLOCK, 0)) != -1)
    {
      int yes = 1;
      BIO *bio;

      // Make sure we can use the same end-point on server side.
      if (!setsockopt(client_fd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes)) &&

          // Bind the server side.
          !bind(client_fd, (struct sockaddr *)&peer, peer_len) &&

          // Connect the peer side.
          !connect(client_fd, (struct sockaddr *)&bound_to, bound_to_len) &&

          // Create a new BIO attachable to the SSL context.
          (bio = BIO_new_dgram(main_fd, BIO_NOCLOSE)))
        {
          // Throwaway sockaddr for DTLS.
          struct sockaddr_storage tmp;
          int ret;

          // Attach the BIO with main_fd to SSL.
          SSL_set_bio(ssl, bio, bio);

          ret = DTLSv1_listen(ssl, (struct sockaddr *)&tmp);

          // Now replace the main_fd with client_fd to keep main_fd safe.
          BIO_set_fd(bio, client_fd, BIO_CLOSE);
          BIO_ctrl(bio, BIO_CTRL_DGRAM_SET_CONNECTED, 0, &peer);

          // Handle SSL_ERROR_WANT_READ or SSL_ERROR_WANT_WRITE.
          return ret;
        }
      close(client_fd);
    }
  return -1; // Conflates with SSL return values if used in real code.
}
```

Very boiled down to show only the necessary steps. For a working and properly
error handled version I wrote the small [DTLS server library](https://github.com/tlsalmin/sukat_dtls "DTLS sukat library"). It's got an easy to use netcat clone.

In summary: Openssl DTLS is usable with non-blocking UDP socket. It just needs
a few tricks to make it work. Quite many examples online have the basic single
client handshake working, but multiple clients connecting fail or show undefined
behaviour.
