================
fedora.tg.client
================
:Authors: Toshio Kuratomi
          Luke Macken
:Date: 2 April 2008
:For Version: 0.3.x

The client module allows you to easily code an application that talks to a
`Fedora Service`_.  It handles the details of decoding the data sent from the
Service into a python data structure and raises an Exception if an error is
encountered.

.. _`Fedora Service`: service.html

.. contents::

----------
BaseClient
----------

The BaseClient class is the basis of all your interactions with the server.
It is flexible enough to be used as is for talking with a service but is
really meant to be subclassed and have methods written for it that do the
things you specifically need to interact with the `Fedora Service`_ you care
about.  Authors of a `Fedora Service`_ are encouraged to provide their own
subclasses of BaseClient that make it easier for other people to use a
particular Service out of the box.

Using Standalone
================

If you don't want to subclass, using BaseClient should be as simple as::

    from fedora.client import BaseClient, AppError, ServerError

    client = BaseClient('https://admin.fedoraproject.org/pkgdb')
    try:
        collectionData = client.send_request('/collections', auth=False)
    except ServerError, e:
        print '%s' % e
    except AppError, e:
        print '%s: %s' % (e.exc, e.message)

    for collection in collectionData['collections']:
        print collection['name'], collection['version']

First you import the BaseClient_ and exceptions from the fedora.client module.

There is now a base class for creating a client application that can talk to
a TurboGears server.  The TurboGears server may need a little bit of tweaking
to make it work with the client.  Please see::

  http://fedorahosted.org/packagedb/wiki/CommandLineClient

for details of how the server may need to be modified.

As an example, let's say you have a TurboGears server that is setup to echo a
message that you send to it.  Additionally, the server requires that you are
logged in in order to access it.  Here's how you access the server from a web
browser::

  $ lynx http://localhost:8080/echo_server/echo/?message='This is a test'

The server will first send you to a login screen where you need to lgin with a
username and password.  Then you will see a web page with 'This is a test'
appearing on the page as your message.

Here's how you might do this using the tg client::

  import getpass
  import sys
  from fedora.tg.client import BaseClient, AuthError

  # Subclass BaseClient and add the methods you need to interact with the server
  class MyClient(BaseClient):
      def echo(self, message):
          # send_request() is the workhorse of BaseClient, sending the data
          # over the wire to the server.
          # The first argument is the server's method name.
          # auth determines whether this request has to send authentication
          #   tokens.  If tokens exist (a cookie) on the filesystem, it will be
          #   used.  If not, it will use a username and password.
          # input is the data to send to the server
          data = self.send_request('echo', auth=True, input={'message':message})
          return data['echo_string']

  if __name__ == '__main__':
      username = 'toshio'
      password = 'XXXX'
      # BASEURL is the base to which the server methods are appended.
      # In this example, the echo method is at:
      #   http://localhost:8080/echo_server/echo
      BASEURL = 'http://localhost:8080/echo_server/'
      client = MyClient(BASEURL, username, password)
      
      # Allow 3 password failures
      for retry in range(0, 3):
          try:
              print client.echo('This is a test')
          except AuthError, e:
              # If authentication fails we get an AuthError.  We take a password
              # from the userand try again.
              if sys.stdin.isatty():
                  # If this is an interactive session, use getpass
                  client.password = getpass.getpass('Password: ')
              else:
                  # If this is part of a script, only try once to read the
                  # password from stdin.  Then fail.
                  if retry > 0:
                      break
                  client.password = sys.stdin.readline().rstrip()