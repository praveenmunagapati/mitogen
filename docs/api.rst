
API Reference
*************


Package Layout
==============


mitogen Package
---------------

.. automodule:: mitogen

.. autodata:: mitogen.is_master
.. autodata:: mitogen.context_id
.. autodata:: mitogen.parent_id
.. autodata:: mitogen.parent_ids


mitogen.core
------------

.. module:: mitogen.core

This module implements most package functionality, but remains separate from
non-essential code in order to reduce its size, since it is also serves as the
bootstrap implementation sent to every new slave context.

.. currentmodule:: mitogen.core
.. decorator:: takes_econtext

    Decorator that marks a function or class method to automatically receive a
    kwarg named `econtext`, referencing the
    :py:class:`mitogen.core.ExternalContext` active in the context in which the
    function is being invoked in. The decorator is only meaningful when the
    function is invoked via :py:data:`CALL_FUNCTION
    <mitogen.core.CALL_FUNCTION>`.

    When the function is invoked directly, `econtext` must still be passed to
    it explicitly.

.. currentmodule:: mitogen.core
.. decorator:: takes_router

    Decorator that marks a function or class method to automatically receive a
    kwarg named `router`, referencing the :py:class:`mitogen.core.Router`
    active in the context in which the function is being invoked in. The
    decorator is only meaningful when the function is invoked via
    :py:data:`CALL_FUNCTION <mitogen.core.CALL_FUNCTION>`.

    When the function is invoked directly, `router` must still be passed to it
    explicitly.


mitogen.master
--------------

.. module:: mitogen.master

This module implements functionality required by master processes, such as
starting new contexts via SSH. Its size is also restricted, since it must
be sent to any context that will be used to establish additional child
contexts.


.. currentmodule:: mitogen.master

.. class:: Select (receivers=(), oneshot=True)

    Support scatter/gather asynchronous calls and waiting on multiple
    receivers, channels, and sub-Selects. Accepts a sequence of
    :py:class:`mitogen.core.Receiver` or :py:class:`mitogen.master.Select`
    instances and returns the first value posted to any receiver or select.

    If `oneshot` is ``True``, then remove each receiver as it yields a result;
    since :py:meth:`__iter__` terminates once the final receiver is removed,
    this makes it convenient to respond to calls made in parallel:

    .. code-block:: python

        total = 0
        recvs = [c.call_async(long_running_operation) for c in contexts]

        for recv, (msg, data) in mitogen.master.Select(recvs):
            print 'Got %s from %s' % (data, recv)
            total += data

        # Iteration ends when last Receiver yields a result.
        print 'Received total %s from %s receivers' % (total, len(recvs))

    :py:class:`Select` may drive a long-running scheduler:

    .. code-block:: python

        with mitogen.master.Select(oneshot=False) as select:
            while running():
                for recv, (msg, data) in select:
                    process_result(recv.context, msg.unpickle())
                for context, workfunc in get_new_work():
                    select.add(context.call_async(workfunc))

    :py:class:`Select` may be nested:

    .. code-block:: python

        subselects = [
            mitogen.master.Select(get_some_work()),
            mitogen.master.Select(get_some_work()),
            mitogen.master.Select([
                mitogen.master.Select(get_some_work()),
                mitogen.master.Select(get_some_work())
            ])
        ]

        for recv, (msg, data) in mitogen.master.Select(selects):
            print data

    .. py:method:: get (timeout=None)

        Fetch the next available value from any receiver, or raise
        :py:class:`mitogen.core.TimeoutError` if no value is available within
        `timeout` seconds.

        :param float timeout:
            Timeout in seconds.

        :return:
            `(receiver, (msg, data))`

    .. py:method:: __bool__ ()

        Return ``True`` if any receivers are registered with this select.

    .. py:method:: close ()

        Remove the select's notifier function from each registered receiver.
        Necessary to prevent memory leaks in long-running receivers. This is
        called automatically when the Python :keyword:`with` statement is used.

    .. py:method:: empty ()

        Return ``True`` if calling :py:meth:`get` would block.

        As with :py:class:`Queue.Queue`, ``True`` may be returned even though a
        subsequent call to :py:meth:`get` will succeed, since a message may be
        posted at any moment between :py:meth:`empty` and :py:meth:`get`.

        :py:meth:`empty` may return ``False`` even when :py:meth:`get` would
        block if another thread has drained a receiver added to this select.
        This can be avoided by only consuming each receiver from a single
        thread.

    .. py:method:: __iter__ (self)

        Yield the result of :py:meth:`get` until no receivers remain in the
        select, either because `oneshot` is ``True``, or each receiver was
        explicitly removed via :py:meth:`remove`.

    .. py:method:: add (recv)

        Add the :py:class:`mitogen.core.Receiver` or
        :py:class:`mitogen.core.Channel` `recv` to the select.

    .. py:method:: remove (recv)

        Remove the :py:class:`mitogen.core.Receiver` or
        :py:class:`mitogen.core.Channel` `recv` from the select. Note that if
        the receiver has notified prior to :py:meth:`remove`, then it will
        still be returned by a subsequent :py:meth:`get`. This may change in a
        future version.


mitogen.fakessh
---------------

.. module:: mitogen.fakessh

fakessh is a stream implementation that starts a local subprocess with its
environment modified such that ``PATH`` searches for `ssh` return an mitogen
implementation of the SSH command. When invoked, this tool arranges for the
command line supplied by the calling program to be executed in a context
already established by the master process, reusing the master's (possibly
proxied) connection to that context.

This allows tools like `rsync` and `scp` to transparently reuse the connections
and tunnels already established by the host program to connect to a target
machine, without wasteful redundant SSH connection setup, 3-way handshakes, or
firewall hopping configurations, and enables these tools to be used in
impossible scenarios, such as over `sudo` with ``requiretty`` enabled.

The fake `ssh` command source is written to a temporary file on disk, and
consists of a copy of the :py:mod:`mitogen.core` source code (just like any
other child context), with a line appended to cause it to connect back to the
host process over an FD it inherits. As there is no reliance on an existing
filesystem file, it is possible for child contexts to use fakessh.

As a consequence of connecting back through an inherited FD, only one SSH
invocation is possible, which is fine for tools like `rsync`, however in future
this restriction will be lifted.

Sequence:

    1. ``fakessh`` Context and Stream created by parent context. The stream's
       buffer has a :py:func:`_fakessh_main` :py:data:`CALL_FUNCTION
       <mitogen.core.CALL_FUNCTION>` enqueued.
    2. Target program (`rsync/scp/sftp`) invoked, which internally executes
       `ssh` from ``PATH``.
    3. :py:mod:`mitogen.core` bootstrap begins, recovers the stream FD
       inherited via the target program, established itself as the fakessh
       context.
    4. :py:func:`_fakessh_main` :py:data:`CALL_FUNCTION
       <mitogen.core.CALL_FUNCTION>` is read by fakessh context,

        a. sets up :py:class:`IoPump` for stdio, registers
           stdin_handle for local context.
        b. Enqueues :py:data:`CALL_FUNCTION <mitogen.core.CALL_FUNCTION>` for
           :py:func:`_start_slave` invoked in target context,

            i. the program from the `ssh` command line is started
            ii. sets up :py:class:`IoPump` for `ssh` command line process's
                stdio pipes
            iii. returns `(control_handle, stdin_handle)` to
                 :py:func:`_fakessh_main`

    5. :py:func:`_fakessh_main` receives control/stdin handles from from
       :py:func:`_start_slave`,

        a. registers remote's stdin_handle with local :py:class:`IoPump`.
        b. sends `("start", local_stdin_handle)` to remote's control_handle
        c. registers local :py:class:`IoPump` with
           :py:class:`mitogen.core.Broker`.
        d. loops waiting for `local stdout closed && remote stdout closed`

    6. :py:func:`_start_slave` control channel receives `("start", stdin_handle)`,

        a. registers remote's stdin_handle with local :py:class:`IoPump`
        b. registers local :py:class:`IoPump` with
           :py:class:`mitogen.core.Broker`.
        c. loops waiting for `local stdout closed && remote stdout closed`


.. currentmodule:: mitogen.fakessh
.. function:: run (dest, router, args, daedline=None, econtext=None)

    Run the command specified by the argument vector `args` such that ``PATH``
    searches for SSH by the command will cause its attempt to use SSH to
    execute a remote program to be redirected to use mitogen to execute that
    program using the context `dest` instead.

    :param mitogen.core.Context dest:
        The destination context to execute the SSH command line in.

    :param mitogen.core.Router router:

    :param list[str] args:
        Command line arguments for local program, e.g.
        ``['rsync', '/tmp', 'remote:/tmp']``

    :returns:
        Exit status of the child process.


Router Class
============


.. currentmodule:: mitogen.core

.. class:: Router

    Route messages between parent and child contexts, and invoke handlers
    defined on our parent context. :py:meth:`Router.route() <route>` straddles
    the :py:class:`Broker <mitogen.core.Broker>` and user threads, it is safe
    to call anywhere.

    **Note:** This is the somewhat limited core version of the Router class
    used by child contexts. The master subclass is documented below this one.

    .. method:: stream_by_id (dst_id)

        Return the :py:class:`mitogen.core.Stream` that should be used to
        communicate with `dst_id`. If a specific route for `dst_id` is not
        known, a reference to the parent context's stream is returned.

    .. method:: add_route (target_id, via_id)

        Arrange for messages whose `dst_id` is `target_id` to be forwarded on
        the directly connected stream for `via_id`. This method is called
        automatically in response to ``ADD_ROUTE`` messages, but remains public
        for now while the design has not yet settled, and situations may arise
        where routing is not fully automatic.

    .. method:: register (context, stream)

        Register a new context and its associated stream, and add the stream's
        receive side to the I/O multiplexer. This This method remains public
        for now while hte design has not yet settled.

    .. method:: add_handler (fn, handle=None, persist=True, respondent=None)

        Invoke `fn(msg)` for each Message sent to `handle` from this context.
        Unregister after one invocation if `persist` is ``False``. If `handle`
        is ``None``, a new handle is allocated and returned.

        :param int handle:
            If not ``None``, an explicit handle to register, usually one of the
            ``mitogen.core.*`` constants. If unspecified, a new unused handle
            will be allocated.

        :param bool persist:
            If ``False``, the handler will be unregistered after a single
            message has been received.

        :param mitogen.core.Context respondent:
            Context that messages to this handle are expected to be sent from.
            If specified, arranges for :py:data:`_DEAD` to be delivered to `fn`
            when disconnection of the context is detected.

            In future `respondent` will likely also be used to prevent other
            contexts from sending messages to the handle.

        :return:
            `handle`, or if `handle` was ``None``, the newly allocated handle.

    .. method:: _async_route(msg, stream=None)

        Arrange for `msg` to be forwarded towards its destination. If its
        destination is the local context, then arrange for it to be dispatched
        using the local handlers.

        This is a lower overhead version of :py:meth:`route` that may only be
        called from the I/O multiplexer thread.

        :param mitogen.core.Stream stream:
            If not ``None``, a reference to the stream the message arrived on.
            Used for performing source route verification, to ensure sensitive
            messages such as ``CALL_FUNCTION`` arrive only from trusted
            contexts.

	.. method:: route(msg)

        Arrange for the :py:class:`Message` `msg` to be delivered to its
        destination using any relevant downstream context, or if none is found,
        by forwarding the message upstream towards the master context. If `msg`
        is destined for the local context, it is dispatched using the handles
        registered with :py:meth:`add_handler`.

        This may be called from any thread.


.. currentmodule:: mitogen.master

.. class:: Router

    Extend :py:class:`mitogen.core.Router` with functionality useful to
    masters, and child contexts who later become masters. Currently when this
    class is required, the target context's router is upgraded at runtime.

    .. note::

        You may construct as many routers as desired, and use the same broker
        for multiple routers, however usually only one broker and router need
        exist. Multiple routers may be useful when dealing with separate trust
        domains, for example, manipulating infrastructure belonging to separate
        customers or projects.

    .. data:: profiling

        When enabled, causes the broker thread and any subsequent broker and
        main threads existing in any child to write
        ``/tmp/mitogen.stats.<pid>.<thread_name>.log`` containing a
        :py:mod:`cProfile` dump on graceful exit. Must be set prior to any
        :py:class:`Broker` being constructed, e.g. via:

        .. code::

             mitogen.master.Router.profiling = True

    .. method:: enable_debug

        Cause this context and any descendant child contexts to write debug
        logs to /tmp/mitogen.<pid>.log.

    .. method:: allocate_id

        Arrange for a unique context ID to be allocated and associated with a
        route leading to the active context. In masters, the ID is generated
        directly, in children it is forwarded to the master via an
        ``ALLOCATE_ID`` message that causes the master to emit matching
        ``ADD_ROUTE`` messages prior to replying.

    .. method:: context_by_id (context_id, via_id=None)

        Messy factory/lookup function to find a context by its ID, or construct
        it. In future this will be replaced by a much more sensible interface.

    .. _context-factories:

    **Context Factories**

    .. method:: local (remote_name=None, python_path=None, debug=False, profiling=False, via=None)

        Arrange for a context to be constructed on the local machine, as an
        immediate subprocess of the current process. The associated stream
        implementation is :py:class:`mitogen.master.Stream`.

        :param str remote_name:
            The ``argv[0]`` suffix for the new process. If `remote_name` is
            ``test``, the new process ``argv[0]`` will be ``mitogen:test``.

            If unspecified, defaults to ``<username>@<hostname>:<pid>``.

        :param str python_path:
            Path to the Python interpreter to use for bootstrap. Defaults to
            ``python2.7``. In future this may default to ``sys.executable``.

        :param bool debug:
            If ``True``, arrange for debug logging (:py:meth:`enable_debug`) to
            be enabled in the new context. Automatically ``True`` when
            :py:meth:`enable_debug` has been called, but may be used
            selectively otherwise.

        :param bool profiling:
            If ``True``, arrange for profiling (:py:data:`profiling`) to be
            enabled in the new context. Automatically ``True`` when
            :py:data:`profiling` is ``True``, but may be used selectively
            otherwise.

        :param mitogen.core.Context via:
            If not ``None``, arrange for construction to occur via RPCs made to
            the context `via`, and for :py:data:`ADD_ROUTE
            <mitogen.core.ADD_ROUTE>` messages to be generated as appropriate.

            .. code-block:: python

                # SSH to the remote machine.
                remote_machine = router.ssh(hostname='mybox.com')

                # Use the SSH connection to create a sudo connection.
                remote_root = router.sudo(username='root', via=remote_machine)

    .. method:: sudo (username=None, sudo_path=None, password=None, \**kwargs)

        Arrange for a context to be constructed over a ``sudo`` invocation. The
        ``sudo`` process is started in a newly allocated pseudo-terminal, and
        supports typing interactive passwords.

        Accepts all parameters accepted by :py:meth:`local`, in addition to:

        :param str username:
            The ``sudo`` username; defaults to ``root``.

        :param str sudo_path:
            Absolute or relative path to ``sudo``. Defaults to ``sudo``.
        :param str password:
            Password to type if/when ``sudo`` requests it. If not specified and
            a password is requested, :py:class:`mitogen.sudo.PasswordError` is
            raised.

    .. method:: ssh (hostname, username=None, ssh_path=None, port=None, check_host_keys=True, password=None, identity_file=None, \**kwargs)

        Arrange for a context to be constructed over a ``ssh`` invocation. The
        ``ssh`` process is started in a newly allocated pseudo-terminal, and
        supports typing interactive passwords.

        Accepts all parameters accepted by :py:meth:`local`, in addition to:

        :param str username:
            The SSH username; default is unspecified, which causes SSH to pick
            the username to use.
        :param str ssh_path:
            Absolute or relative path to ``ssh``. Defaults to ``ssh``.
        :param int port:
            Port number to connect to; default is unspecified, which causes SSH
            to pick the port number.
        :param bool check_host_keys:
            If ``False``, arrange for SSH to perform no verification of host
            keys. If ``True``, cause SSH to pick the default behaviour, which
            is usually to verify host keys.
        :param str password:
            Password to type if/when ``ssh`` requests it. If not specified and
            a password is requested, :py:class:`mitogen.ssh.PasswordError` is
            raised.
        :param str identity_file:
            Path to an SSH private key file to use for authentication. Default
            is unspecified, which causes SSH to pick the identity file.

            When this option is specified, only `identity_file` will be used by
            the SSH client to perform authenticaion; agent authentication is
            automatically disabled, as is reading the default private key from
            ``~/.ssh/id_rsa``, or ``~/.ssh/id_dsa``.


Context Class
=============

.. currentmodule:: mitogen.core

.. class:: Context

    Represent a remote context regardless of connection method.

    **Note:** This is the somewhat limited core version of the Context class
    used by child contexts. The master subclass is documented below this one.

    .. method:: send (msg)

        Arrange for `msg` to be delivered to this context. Updates the
        message's `dst_id` prior to routing it via the associated router.

        :param mitogen.core.Message msg:
            The message.

    .. method:: send_async (msg, persist=False)

        Arrange for `msg` to be delivered to this context, with replies
        delivered to a newly constructed Receiver. Updates the message's
        `dst_id` prior to routing it via the associated router and registers a
        handle which is placed in the message's `reply_to`.

        :param bool persist:
            If ``False``, the handler will be unregistered after a single
            message has been received.

        :param mitogen.core.Message msg:
            The message.

        :returns:
            :py:class:`mitogen.core.Receiver` configured to receive any replies
            sent to the message's `reply_to` handle.

    .. method:: send_await (msg, deadline=None)

        As with :py:meth:`send_async`, but expect a single reply
        (`persist=False`) delivered within `deadline` seconds.

        :param mitogen.core.Message msg:
            The message.

        :param float deadline:
            If not ``None``, seconds before timing out waiting for a reply.

        :raises mitogen.core.TimeoutError:
            No message was received and `deadline` passed.


.. currentmodule:: mitogen.master

.. class:: Context

    Extend :py:class:`mitogen.core.Router` with functionality useful to
    masters, and child contexts who later become masters. Currently when this
    class is required, the target context's router is upgraded at runtime.

    .. method:: call_async (fn, \*args, \*\*kwargs)

        Arrange for the context's ``CALL_FUNCTION`` handle to receive a
        message that causes `fn(\*args, \**kwargs)` to be invoked on the
        context's main thread.

        :param fn:
            A free function in module scope, or a classmethod or staticmethod
            of a class directly reachable from module scope:

            .. code-block:: python

                # mymodule.py

                def my_func():
                    """A free function reachable as mymodule.my_func"""

                class MyClass:
                    @staticmethod
                    def my_staticmethod():
                        """Reachable as mymodule.MyClass.my_staticmethod"""

                    @classmethod
                    def my_classmethod(cls):
                        """Reachable as mymodule.MyClass.my_classmethod"""

                    def my_instancemethod(self):
                        """Unreachable: requires a class instance!"""

                    class MyEmbeddedClass:
                        @classmethod
                        def my_classmethod(cls):
                            """Not directly reachable from module scope!"""

        :param tuple args:
            Function arguments, if any. See :ref:`serialization-rules` for
            permitted types.
        :param dict kwargs:
            Function keyword arguments, if any. See :ref:`serialization-rules`
            for permitted types.
        :returns:
            :py:class:`mitogen.core.Receiver` configured to receive the result
            of the invocation:

            .. code-block:: python

                recv = context.call_async(os.check_output, 'ls /tmp/')
                try:
                    print recv.get_data()  # Prints output once it is received.
                except mitogen.core.CallError, e:
                    print 'Call failed:', str(e)

            Asynchronous calls may be dispatched in parallel to multiple
            contexts and consumed as they complete using
            :py:class:`mitogen.master.Select`.

    .. method:: call (fn, \*args, \*\*kwargs)

        Equivalent to :py:meth:`call_async(fn, \*args, \**kwargs).get_data()
        <call_async>`.

        :returns:
            The function's return value.

        :raises mitogen.core.CallError:
            An exception was raised in the remote context during execution.



Receiver Class
--------------

.. currentmodule:: mitogen.core

.. class:: Receiver (router, handle=None, persist=True, respondent=None)

    Receivers are used to wait for pickled responses from another context to be
    sent to a handle registered in this context. A receiver may be single-use
    (as in the case of :py:meth:`mitogen.master.Context.call_async`) or
    multiple use.

    :param mitogen.core.Router router:
        Router to register the handler on.

    :param int handle:
        If not ``None``, an explicit handle to register, otherwise an unused
        handle is chosen.

    :param bool persist:
        If ``True``, do not unregister the receiver's handler after the first
        message.

    :param mitogen.core.Context respondent:
        Reference to the context this receiver is receiving from. If not
        ``None``, arranges for the receiver to receive :py:data:`_DEAD` if
        messages can no longer be routed to the context, due to disconnection
        or exit.

    .. attribute:: notify = None

        If not ``None``, a reference to a function invoked as
        `notify(receiver)` when a new message is delivered to this receiver.
        Used by :py:class:`mitogen.master.Select` to implement waiting on
        multiple receivers.

    .. py:method:: empty ()

        Return ``True`` if calling :py:meth:`get` would block.

        As with :py:class:`Queue.Queue`, ``True`` may be returned even though a
        subsequent call to :py:meth:`get` will succeed, since a message may be
        posted at any moment between :py:meth:`empty` and :py:meth:`get`.

        :py:meth:`empty` is only useful to avoid a race while installing
        :py:attr:`notify`:

        .. code-block:: python

            recv.notify = _my_notify_function
            if not recv.empty():
                _my_notify_function(recv)

            # It is guaranteed the receiver was empty after the notification
            # function was installed, or that it was non-empty and the
            # notification function was invoked at least once.

    .. py:method:: close ()

        Cause :py:class:`mitogen.core.ChannelError` to be raised in any thread
        waiting in :py:meth:`get` on this receiver.

    .. py:method:: get (timeout=None)

        Sleep waiting for a message to arrive on this receiver.

        :param float timeout:
            If not ``None``, specifies a timeout in seconds.

        :raises mitogen.core.ChannelError:
            The remote end indicated the channel should be closed, or
            communication with its parent context was lost.

        :raises mitogen.core.TimeoutError:
            Timeout was reached.

        :returns:
            `(msg, data)` tuple, where `msg` is the
            :py:class:`mitogen.core.Message` that was received, and `data` is
            its unpickled data part.

    .. py:method:: get_data (timeout=None)

        Like :py:meth:`get`, except only return the data part.

    .. py:method:: __iter__ ()

        Block and yield `(msg, data)` pairs delivered to this receiver until
        :py:class:`mitogen.core.ChannelError` is raised.


Sender Class
------------

.. currentmodule:: mitogen.core

.. class:: Sender (context, dst_handle)

    Senders are used to send pickled messages to a handle in another context,
    it is the inverse of :py:class:`mitogen.core.Sender`.

    :param mitogen.core.Context context:
        Context to send messages to.
    :param int dst_handle:
        Destination handle to send messages to.

    .. py:method:: close ()

        Send :py:data:`_DEAD` to the remote end, causing
        :py:meth:`ChannelError` to be raised in any waiting thread.

    .. py:method:: put (data)

        Send `data` to the remote end.


Channel Class
-------------

.. currentmodule:: mitogen.core

.. class:: Channel (router, context, dst_handle, handle=None)

    A channel inherits from :py:class:`mitogen.core.Sender` and
    `mitogen.core.Receiver` to provide bidirectional functionality.

    Since all handles aren't known until after both ends are constructed, for
    both ends to communicate through a channel, it is necessary for one end to
    retrieve the handle allocated to the other and reconfigure its own channel
    to match. Currently this is a manual task.


Broker Class
============

.. currentmodule:: mitogen.core
.. class:: Broker

    Responsible for handling I/O multiplexing in a private thread.

    **Note:** This is the somewhat limited core version of the Broker class
    used by child contexts. The master subclass is documented below this one.

    .. attribute:: shutdown_timeout = 3.0

        Seconds grace to allow :py:class:`streams <Stream>` to shutdown
        gracefully before force-disconnecting them during :py:meth:`shutdown`.

    .. method:: defer (func, \*args, \*kwargs)

        Arrange for `func(\*args, \**kwargs)` to be executed on the broker
        thread, or immediately if the current thread is the broker thread. Safe
        to call from any thread.

    .. method:: start_receive (stream)

        Mark the :py:attr:`receive_side <Stream.receive_side>` on `stream` as
        ready for reading. Safe to call from any thread. When the associated
        file descriptor becomes ready for reading,
        :py:meth:`BasicStream.on_receive` will be called.

    .. method:: stop_receive (stream)

        Mark the :py:attr:`receive_side <Stream.receive_side>` on `stream` as
        not ready for reading. Safe to call from any thread.

    .. method:: start_transmit (stream)

        Mark the :py:attr:`transmit_side <Stream.transmit_side>` on `stream` as
        ready for writing. Safe to call from any thread. When the associated
        file descriptor becomes ready for writing,
        :py:meth:`BasicStream.on_transmit` will be called.

    .. method:: stop_receive (stream)

        Mark the :py:attr:`transmit_side <Stream.receive_side>` on `stream` as
        not ready for writing. Safe to call from any thread.

    .. method:: shutdown

        Request broker gracefully disconnect streams and stop.

    .. method:: join

        Wait for the broker to stop, expected to be called after
        :py:meth:`shutdown`.

    .. method:: keep_alive

        Return ``True`` if any reader's :py:attr:`Side.keep_alive` attribute is
        ``True``, or any :py:class:`Context` is still registered that is not
        the master. Used to delay shutdown while some important work is in
        progress (e.g. log draining).

    **Internal Methods**

    .. method:: _broker_main

        Handle events until :py:meth:`shutdown`. On shutdown, invoke
        :py:meth:`Stream.on_shutdown` for every active stream, then allow up to
        :py:attr:`shutdown_timeout` seconds for the streams to unregister
        themselves before forcefully calling
        :py:meth:`Stream.on_disconnect`.


.. currentmodule:: mitogen.master
.. class:: Broker

    .. note::

        You may construct as many brokers as desired, and use the same broker
        for multiple routers, however usually only one broker need exist.
        Multiple brokers may be useful when dealing with sets of children with
        differing lifetimes. For example, a subscription service where
        non-payment results in termination for one customer.

    .. attribute:: shutdown_timeout = 5.0

        Seconds grace to allow :py:class:`streams <Stream>` to shutdown
        gracefully before force-disconnecting them during :py:meth:`shutdown`.


Utility Functions
=================

.. module:: mitogen.utils

A random assortment of utility functions useful on masters and children.

.. currentmodule:: mitogen.utils
.. function:: disable_site_packages

    Remove all entries mentioning ``site-packages`` or ``Extras`` from the
    system path. Used primarily for testing on OS X within a virtualenv, where
    OS X bundles some ancient version of the :py:mod:`six` module.

.. currentmodule:: mitogen.utils
.. function:: log_to_file (path=None, io=True, level='INFO')

    Install a new :py:class:`logging.Handler` writing applications logs to the
    filesystem. Useful when debugging slave IO problems.

    :param str path:
        If not ``None``, a filesystem path to write logs to. Otherwise, logs
        are written to :py:data:`sys.stderr`.

    :param bool io:
        If ``True``, include extremely verbose IO logs in the output. Useful
        for debugging hangs, less useful for debugging application code.

    :param str level:
        Name of the :py:mod:`logging` package constant that is the minimum
        level to log at. Useful levels are ``DEBUG``, ``INFO``, ``WARNING``,
        and ``ERROR``.

.. currentmodule:: mitogen.utils
.. function:: run_with_router(func, \*args, \**kwargs)

    Arrange for `func(router, \*args, \**kwargs)` to run with a temporary
    :py:class:`mitogen.master.Router`, ensuring the Router and Broker are
    correctly shut down during normal or exceptional return.

    :returns:
        `func`'s return value.

.. currentmodule:: mitogen.utils
.. decorator:: with_router

    Decorator version of :py:func:`run_with_router`. Example:

    .. code-block:: python

        @with_router
        def do_stuff(router, arg):
            pass

        do_stuff(blah, 123)


Exceptions
==========

.. currentmodule:: mitogen.core

.. class:: Error (fmt, \*args)

    Base for all exceptions raised by Mitogen.

.. class:: CallError (e)

    Raised when :py:meth:`Context.call() <mitogen.master.Context.call>` fails.
    A copy of the traceback from the external context is appended to the
    exception message.

.. class:: ChannelError (fmt, \*args)

    Raised when a channel dies or has been closed.

.. class:: StreamError (fmt, \*args)

    Raised when a stream cannot be established.

.. class:: TimeoutError (fmt, \*args)

    Raised when a timeout occurs on a stream.
