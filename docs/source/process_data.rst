=====================
Process Data Exchange
=====================

Warning: Properly handling EtherCAT process data with Python is difficult,
especially when working with distributed clocks (DC). Please read the full section before attempting.

In theory, reading and writing process data with PYSOEM is simple:

.. code-block:: python

   while True:
      master.send_processdata()
      actual_wkc = master.receive_processdata(timeout=100_000)

      # Loop at the network cycle rate (10ms).
      time.sleep(0.01)

However, if you run this code you'll find it eventually results in network errors and/or missed process data.
The problem is that with distributed clocks, the global network clock is based from the clock on the first slave device,
which is far more accurate than the computer with a non-realtime OS running Python.

The difficult part about EtherCAT process data is that slaves expect new process data
at highly accurate intervals; if a network cycle (or cycles) are missed, watchdog timers on the slaves
will begin to expire and throw errors. When you put an EtherCAT connection into OP state,
you're effectively signing a contract that new process data will be delivered on time, always,
until OP state is exited. This means we need to get the timing very reliable.

If you look at the code above carefully, you'll notice that :py:meth:`~pysoem.Master.send_processdata` isn't actually called every 10 milliseconds;
it's called every 10 milliseconds *plus the time required for the send & recieve function calls*.
Additionally, sleep functions in Python are highly inaccurate (and OS dependent).

A more accurate approach
----------------------------------

A better approach for exchanging process data is as follows:

1. Start a timer.
2. Send/recieve process data.
3. Measure the network DC time with :py:meth:`~pysoem.Master.dc_time`.
4. Measure the timer to determine how much time has elapsed in the loop.
5. Sleep for the remainder of the network cycle, offset the by error from the global network time.

This approach works much better. It accounts for delays due to Python calls within the process data loop,
and accounts for drift between the master and global DC time.

Tip: Use Python's :py:meth:`time.perf_counter_ns` for the most accurate system timer.

Problems with sleeping
----------------------------------
