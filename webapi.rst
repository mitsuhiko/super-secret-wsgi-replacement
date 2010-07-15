Python Web API 1.0
==================

This document specifies a proposed standard interface between web servers
and Python web frameworks, to promote web application portability across a
variety of web servers.  It acts as replacement for the previous
:pep:`333` aka WSGI 1.0.

Rationale and Goals
-------------------

WSGI was a huge success in the Python community.  Not only in the Python
community, similar specifications appeared for other programming languages
as well.  However with WSGI being adopted everywhere it also become
obvious that the specification was lacking in many situations.  The latest
in a series of limitations became obvious when Python 3 started adopting
WSGI and implementing it in a way the PEP original specified.

The goal of this specification is to fix the issues with WSGI by both
being one level higher than WSGI and at the same time giving access to
features people previously wanted from WSGI that were unavailable.

Specification Overview
----------------------

This specification is written with Python 3 in mind.  All code examples
in this document are for Python 3.  A few terms:

native string:
    a native string is a bytestring on Python 2 and an unicode string
    on Python 3.

unicode string:
    ``unicode`` on Python 2, ``str`` on Python 3

bytes:
    ``str`` on Python 2, ``bytes`` or ``bytearray`` on Python 3

server:
    A server is what invokes a callable from the low level API (the
    request handler) and passes it an appropriate formatted dictionary of
    incoming data (the request environment) and then ensures that the
    return value is sent to the client appropriately.  This specification
    defines different kinds of feature sets a server has to implement.

request handler:
    A function that accepts a WSGI environment and returns the response.
    The request handler is a feature of the low level API.  The response
    can either be a response tuple or a callable that returns the
    response tuple once it's ready.  More on that below.

Low Level API
-------------

The low level API is the communication interface between server and
application.  It is modelled after the WSGI 1.0 specification but with
well defined rules for unicode strings and with a few issues resolved.

An "request handler" is a callable that is passed the incoming request
data as `environment` which is a dict that gives access to the incoming
information.  The return value of the callable is a tuple consisting of
headers, headers and body::

    def request_handler(environ):
        data = 'Hello World!'.encode('utf-8')
        return data, (200, 'OK'), [
            ('Content-Type', 'text/plain; charset=utf-8'),
            ('Content-Length', str(len(data)))
        ]

The components:

`environ`
    The environment is a dictionary where the keys are native strings.
    The exact contents are outlined below.  This is called the "request
    environment"

response tuple
    The response tuple is what is returned from the application.  It is a
    tuple of three items in the form ``(appiter, status, headers)``.

appiter
    The appiter is an iterator that yields bytes.  It might be an empty
    iterator if the body should be empty.  Unicode strings are strictly
    forbidden here.  Unlike WSGI iterators, the ``close()`` method on an
    iterator does not have to be called by the server.  However the server
    and each component that might operate on the app iter has to fully
    exhaust it.  This makes it possible to replace the usage of
    ``close()`` with a try/finally block.

status
    The status is the status code for the response as tuple in the form
    ``(code, messages)``.  Where the code is an integer and the message
    can be unicode or bytes.  If unicode is sent to the server it has to
    encode it with the latin1 encoding and raise an `UnicodeError` if this
    fails.

    The status is in the form ``(code, message)``.  All status codes are
    allowed from the API perspective but some web servers might not be
    able to transparently pass all of them back to the browser.  This will
    especially be the case when a webapi application is running on top of
    a WSGI bridge.

headers
    These are the headers send back to the client.  Unlike the previous
    WSGI specification, this does not have to be a list, it might be any
    kind of iterable, including an unsized iterator.  The items of this
    iterator are tuples in the form ``(header, value)`` where both might
    be unicode strings or bytes.  In case of unicode strings, the server
    must encode them with the latin1 encoding and raise an `UnicodeError`
    if this fails.

Request Environment
-------------------

The request environment is a dict with the incoming request data.  The keys
for this dictionary are native strings, the values of different kinds (as
documented):

=============================== =========================================
``webapi.headers``              The incoming request headers as list
                                of tuples in the form ``(key, value)``.
                                Both key and value are native strings,
                                The server has to decode the value as
                                latin1 and use the `replace` error
                                handling for decode errors where native
                                strings are unicode strings.
``webapi.prefix``               The prefix to requests as bytes.  This
                                path is kept URL encoded unlike the CGI
                                `PATH_INFO` header. *
``webapi.path``                 The relative path of the URL below the
                                prefix as bytes.  This path is kept URL
                                encoded unlike the CGI `PATH_INFO`
                                header. *
``webapi.query_string``         The query string part of the URL as
                                bytes.
``webapi.is_secure``            `True` if the request came from an
                                HTTPS request.
``webapi.server_addr``          The name of the server that handles
                                the application and the port
                                ``(server_name, server_port)``.  The
                                server name is a native string, the
                                port an integer **
``webapi.application_path``     The path to the application on the
                                current host. **
``webapi.input``                A stream of incoming request data
``webapi.client_addr``          The address of the client as tuple in
                                the form ``(address, port)``.  The
                                address is a native string, the port is
                                an integer.
``webapi.method``               the HTTP request method as native string.
                                The server has to decode the value as
                                latin1 and use the `replace` error
                                handling for decode errors where native
                                strings are unicode strings.
``webapi.setup_environ``        The setup environment that was passed to
                                the factory function.
=============================== =========================================

Values marked with a star (``*``) are part of the two-level dispatching
which is explained below.  Values marked with two stars (``**``) are
normally provided by the setup environment but might be unavailable at
that time or change for a request.  These values are only present if they
differ from the values in ``webapi.setup_environ``.

The input stream is a file object opened in read only binary more which
does not have to support seeking but all other operations.  The server has
to ensure that a call to ``environ['webapp.input'].read()`` is safe, thus
limiting the incoming data to the content length.

.. TODO: Is there a PSH?  Are there cases of bodies without content length?

Readers familiar with the WSGI specification will notice that some keys
present in WSGI are missing.  Especially there seem to be no keys for
``SCRIPT_NAME`` and ``wsgi.errors`` among the different kinds of
deployment hints such as ``wsgi.run_once``.  The reason for this a request
handler is not just passed to a server, but that this request handler is
returned from a function that is passed a so called "setup environment"
which this information.

Setup Environment
-----------------

The setup environment is a second environment that is passed to the
function that returns the request handler.  This environment contains
deployment specific keys that do not change between requests.  The
following keys are required:

=============================== =========================================
``webapi.version``              The version of the webapp specification
``webapi.compliance_level``     The server compliance level.  See the
                                note below on server compliances for more
                                information.
``webapi.async``                `True` if the server supports deferred
                                responses.
``webapi.multithreaded``        `True` if the server uses multithreading
                                for request handling.
``webapi.multiprocess``         `True` if the server uses multiple
                                processes for request handling.
``webapi.thread_reuse``         `True` if this server has a pool of
                                threads or reuses threads in a different
                                way.
``webapi.process_reuse``        `True` if this server reuses the
                                processes for request handling.
``webapi.server_addr``          The name of the server that handles
                                the application and the port
                                ``(server_name, server_port)``.  The
                                server name is a native string, the
                                port an integer *
``webapi.application_path``     The path to the application on the
                                current host. *
``webapi.prefer_ssl``           `True` if SSL is preferred for the
                                server.  Can be used for URL building.
=============================== =========================================

All keys are required except for keys marked with a star.  If a server is
unable to provide these values at the time the setup environment is set
up, it might pass those in the request environment instead.  In that case
the values in the setup environment must be `None`.  This can be the case
if the server configuration has wildcards activated for subdomains or
applications.

If the server does not know a setting (eg: if threads are reused or not)
it should set the value to `None`.

Low Level Registration
----------------------

In WSGI the application function was passed directly to the WSGI server
and the server executed that function on each request.  In webapi a
factory function is passed to the server instead which is invoked with the
setup environment and returns the request handler.

Here a basic example that accepts the server config and returns a request
handler::

    def app_factory(setup_environ):
        def request_handler(environ):
            rv = 'Server name: %s' % setup_environ['webapi.server_addr'][0]
            data = rv.encode('utf-8')
            return data, (200, 'OK'), [
                ('Content-Type', 'text/plain; charset=utf-8'),
                ('Content-Length', str(len(data)))
            ]
        return request_handler

Because the interface works with any callable, it can also be used to
register classes.  This example works as well and does the same::

    class MyApplication(object):

        def __init__(self, setup_environ):
            self.setup_environ = setup_environ

        def __call__(self, environ):
            rv = 'Server name: %s' % self.setup_environ['webapi.server_addr'][0]
            data = rv.encode('utf-8')
            return data, (200, 'OK'), [
                ('Content-Type', 'text/plain; charset=utf-8'),
                ('Content-Length', str(len(data)))
            ]

Server Compliance Levels
------------------------

Because this specification specifies something that might not be
implementable and certainly not on top of an unmodified WSGI specification
there are different levels of compatibility.  In general the high level
interface will degrade gracefully for all levels, but certain applications
sitting on top of it might not work on all servers.

The following compatibility levels are defined as part of this
specification:

0. The server follows the specification in all respects.  This will usually
   be the case for servers that speak HTTP directly and are proxied.
1. The server is implemented on top of a CGI inspired protocol and might
   not give access to the original values of the request path or all
   incoming headers.  They should try to reconstruct the values as good as
   possible though.
2. Like compliance level 1, but with the additional restriction that the
   ``webapi.server_addr`` or ``webapi.process_reuse`` will be unavailable
   in all situations.  If the server is capable of giving away this
   information ahead of time but due to the configuration cannot provide
   it, it might still be compliant to 1 or 0.
3. The server is running on top of WSGI and has to adhere to the WSGI
   limitations regarding headers.

Middlewares
-----------

WSGI like Middlewares are replaced by request wrappers.  Request wrappers
are invoked like request handlers and return the same responses but
evaluate the request transparently like a server would.  They are free to
buffer any data or defer execution, but they have to follow these rules:

1.  the appiter from the request handler that was passed to the request
    wrapper has to be fully evaluated until the `StopIteration`.
2.  request wrappers must be able to deal with deferred responses (more
    below) but must not return deferred responses unless necessary.
    Necessary means the original responses was deferred already or the
    request wrapper wants to optimize in an async environment.
3.  request wrappers must not read the input stream unless they are
    intended to be used for debugging or testing purposes only
    (interactive debugging middlewares, profilers etc.)

Request wrappers are allowed to perform modifications on the request
environment but they are required to revert the changes after execution!

Two-Level Dispatching
---------------------

One of the problems WSGI was facing is that it many middlewares were
rewriting the environment and it was not obvious for an application where
the actual root of the application is.  In webapi there are two levels of
request dispatching.

The actual root of the application is defined in the setup environment as
``webapi.application_path`` and the name of the server as
``webapi.server_addr``.  If these informations are not available at setup
time they are `None` and transmitted in the request environment.  An
interesting aspect is the server name.  This always referrs to the base
host name of the application.  For example if an application is listening
on ``*.example.com``, the server name would be ``example.com``.  To figure
out where the request actually went, the application can use the ``Host``
HTTP header.  This makes it possible to easily extract the subdomain for
an application.  The ``webapi.application_path`` is where the application
is listening.  This usually is ``/``, but if the server is configured
otherwise it might be ``/app`` or something else.

The per-request information is stored in the request environment as
``webapi.prefix`` and ``webapi.path``.  The prefix replaces the
``webapi.path`` for the request, ``webapi.path`` is what comes after the
prefix.  To clear up the confusion, let's start with the most basic case:

-   the application is mounted at ``/`` on the server ``example.com``
-   in the setup environment the ``webapi.application_path`` is ``/``
    and the ``webapi.server_addr`` is ``('example.com', 80)``
-   a request comes in to ``http://www.example.com/index.html``.
-   in this case the request values are:

    * ``webapi.prefix`` is ``/``
    * ``webapi.path`` is ``index.html``

Now when would ``webapi.prefix`` in the request environment differ from
the ``webapi.application_path`` in the setup environment?  In case a
request wrapper is rewriting the request.  Imagine a request handler
should listen on ``http://example.com/wiki``.  This request handler acts
as a "sub request handler" invoked by another request handler.

-   the application is mounted at ``/`` on the server ``example.com``
-   everything below ``wiki`` is sent to another request handler
-   in the setup environment the ``webapi.application_path`` is ``/``
    and the ``webapi.server_addr`` is ``('example.com', 80)``
-   a request comes in to ``http://www.example.com/wiki/Main_Page``.
-   in this case the request values are:

    * ``webapi.prefix`` is ``/wiki/``
    * ``webapi.path`` is ``index.html``

In the case that a server is unable to provide the
``webapi.application_path``  and ``webapi.server_addr`` in advance to the
application, it must provide these values in the request environment.  If
it does not know the server name at all it must still pass the key, but
set the value to `None`.  It should not try to reconstruct the server name
from the host header, this is up for the application to do.  This give the
application the chance to recognize a server setup without a reliable
server name.

Consuming the Application Iterator
----------------------------------

The application iterator always yields bytes.  The server translates every
iteration into a ``write()`` into a file object on a socket back to the
client.  The server is free to flush whenever it thinks it is necessary
but is required to flush if a bytes object with the length of zero is
recieved.

Pseudocode::

    def make_bytes(s):
        if isinstance(s, str):
            s = s.encode('latin1')
        return s

    def invoke(request_handler):
        rv = request_handler(environ)
        if hasattr(rv, '__call__'):
            raise RuntimeError('this server does not support deferred '
                               'application iterators')

        appiter, status, headers = rv
        f.write(b'HTTP/1.1 ' + make_bytes(status) + '\r\n')
        for key, value in headers:
            f.write(make_bytes(key) + b': ' + make_bytes(value) + '\r\n')
        f.write(b'\r\n')

        for item in appiter:
            if not item:
                f.flush()
            else:
                f.write(item)
        f.close()

.. XXX: connection close?  up to app?

URL Reconstruction
------------------

The URL reconstruction should always take the values from the request
environment into consideration.  And then combine them with the values
from the setup environment.  The server addr for instance can come from
the setup environment but might be `None` there in which case the
algorithm has to look at the values from the current request environment.


Deferred Responses
------------------

If a request handler does not return a response tuple but a callable
object, it has two options:

-   the server supports asyncronous request handling and has to handle the
    return value as handled below.
-   it does not support async execution and did notify the application
    about that in the setup environment by setting ``webapi.async`` to
    `False`.  In that case the server is required to raise an exception
    and abort the request.

In general the idea of the callable is that the server will call it until
it returns a response tuple instead of `None`.  In that case, this data is
sent back straight to the server.  It should still assume that the
response data is not yet fully available like with regular response
processing.  Once `StopIteration` is raised it can close the connection to
the client if the headers say so and clean up.

It is up to the server how it manages the deferred callables.
