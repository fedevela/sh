:orphan:

Completed Async Command Result Architecture
###########################################

Status
======

This record assigns the completed asynchronous result procedures in
``asynchronous_execution_logic.rst`` to the existing single-module runtime.
It introduces no package, process, persistence, configuration, or deployment
boundary.  The implementation must evolve the existing ``sh.py`` ownership
centres and preserve synchronous command behaviour.

Architectural Decision
======================

``RunningCommand`` is the identity and result-policy boundary for an
asynchronous invocation.  ``Command`` continues to own argument resolution and
construction, while ``OProc`` and its output worker continue to own subprocess
state and byte collection.  The existing ``asyncio.Event`` on
``RunningCommand`` is the one-way completion seam from the output worker to
awaiting event-loop tasks.

No new result object or completion future is introduced.  A second object
would weaken the identity guarantee in SHAWAIT-001 and create a second owner
for the process state required by SHAWAIT-003 and SHAWAIT-015.

Loci and Ownership
==================

``Command.__call__`` and ``Command.RunningCommandCls``
-------------------------------------------------------

**Requirements:** SHAWAIT-001, SHAWAIT-004, SHAWAIT-015.

**Verification:**
``test_shawait_001_return_cmd_await_returns_constructed_running_command``,
``test_shawait_004_default_await_returns_fully_decoded_text_output``, and
``test_shawait_015_repeated_await_returns_same_command_without_respawn``.

**Procedure:** ``construct_async_invocation``.

``Command.__call__`` owns the final merge of default, baked, and invocation
special arguments.  Its ``call_args`` mapping is the invocation contract and
the authoritative source for ``async``, ``return_cmd``, ``encoding``,
``decode_errors``, ``ok_code``, and timeout policy.  It constructs exactly one
``RunningCommand`` through ``Command.RunningCommandCls``.  For an asynchronous
call it returns that object without applying the synchronous text-return
branch.

Incoming dependencies are the public command invocation and baked arguments.
Outgoing dependencies are the existing ``RunningCommandCls`` construction seam
and the compiled command passed to it.  ``Command`` owns neither process
completion nor await-time result selection.

``RunningCommand.__init__`` and ``RunningCommand.__await__``
-------------------------------------------------------------

**Requirements:** SHAWAIT-001, SHAWAIT-002, SHAWAIT-003, SHAWAIT-004,
SHAWAIT-015.

**Verification:** all five named ``AsyncAwaitContractTests`` obligations.

**Procedures:** ``construct_async_invocation`` and
``await_completed_async_result``.

``RunningCommand`` owns invocation identity, the retained ``call_args``, the
single ``OProc`` reference, and the await result policy.  Construction creates
one unset ``aio_output_complete`` event when invoked on a running event loop.
``__await__`` consumes that event, delegates finalization to ``wait()``, and
then chooses its result from the retained ``return_cmd`` value:

* true returns ``self``;
* false decodes the completed ``OProc.stdout`` bytes with the retained
  ``encoding`` and ``decode_errors`` values.

The event is a level-triggered completion latch: once set, later awaits pass
immediately.  ``RunningCommand._waited_until_completion`` makes finalization
idempotent.  Together they ensure repeated awaits reuse the same command,
process, PID, bytes, and exit code without re-entering command construction.

Cancellation belongs to the awaiting asyncio task.  It does not transfer
process ownership, clear the event, or create a replacement command; the
existing process and output workers continue and a later await uses the same
completion latch.

``RunningCommand.wait`` and ``OProc.wait``
------------------------------------------

**Requirements:** SHAWAIT-002, SHAWAIT-003, SHAWAIT-015.

**Verification:**
``test_shawait_002_await_stays_pending_until_process_and_output_complete``,
``test_shawait_003_completed_command_exposes_stdout_stderr_and_exit_code``, and
``test_shawait_015_repeated_await_returns_same_command_without_respawn``.

**Procedure:** ``await_completed_async_result``.

``OProc`` owns the child PID, adjusted ``exit_code``, stream readers, captured
stdout and stderr byte deques, and worker lifecycle.  ``OProc.wait`` is the
process and thread cleanup boundary; its wait lock is the sole synchronization
authority for reaping and exit-code mutation.  ``RunningCommand.wait`` is the
user-facing finalization boundary.  It translates timeout and unacceptable
exit status into the existing ``TimeoutException`` and ``ErrorReturnCode``
families after cleanup, and records that completion has already been handled.

No bytes or exit state are copied into a second owner.  The public
``RunningCommand.stdout``, ``stderr``, and ``exit_code`` properties continue to
read the authoritative completed state from their one ``OProc``.

``output_thread`` and the completion callback
---------------------------------------------

**Requirements:** SHAWAIT-002, SHAWAIT-003.

**Verification:**
``test_shawait_002_await_stays_pending_until_process_and_output_complete`` and
``test_shawait_003_completed_command_exposes_stdout_stderr_and_exit_code``.

**Procedure:** ``publish_async_output_completion``.

``output_thread`` owns polling and closing the managed stdout and stderr
``StreamReader`` instances.  The readers append retained bytes to the deques
owned by their ``OProc``.  After polling has ended, the worker observes process
termination, closes both readers, and invokes its completion callback.  For an
async invocation, that callback crosses the thread/event-loop boundary only by
calling ``loop.call_soon_threadsafe(aio_output_complete.set)``.

The completion callback carries no result payload and does not select a return
mode.  Its only contract is to publish that process termination and output
collection have crossed the existing output-completion boundary.  This keeps
the worker independent of the public ``_return_cmd`` policy.

Contracts and Dependency Direction
==================================

The cooperating contracts, in dependency order, are:

#. ``Command.__call__`` produces one mutable invocation contract
   (``call_args``) and one ``RunningCommand``.
#. ``RunningCommand`` retains that contract and owns one ``OProc`` plus one
   asyncio completion event.
#. ``OProc`` owns process state and delegates stream consumption to its worker
   and ``StreamReader`` collaborators.
#. ``output_thread`` publishes completion through the callback supplied by
   ``OProc``; the callback schedules the event transition on the owning event
   loop.
#. ``RunningCommand.__await__`` depends on the event, then on the existing
   ``wait`` boundary, and finally on its retained invocation policy.

Dependencies therefore flow from the public ``Command`` facade toward
``RunningCommand``, then toward ``OProc`` and stream collaborators.  The worker
does not depend on ``Command`` result semantics.  The only reverse notification
is the narrow, payload-free completion callback.  There is no persisted data,
transaction boundary, queue, external event, adapter, authorization boundary,
or migration.

State and Lifecycle
===================

The owning loci preserve this lifecycle::

   Command.__call__
       -> one RunningCommand / one OProc / one unset asyncio.Event
       -> process running while output_thread collects bytes
       -> OProc observes and records process exit
       -> output_thread closes both readers
       -> event-loop callback sets asyncio.Event
       -> RunningCommand.__await__ calls idempotent wait()
       -> timeout or exit-code error, otherwise object-or-text result selection
       -> repeated await reuses the set event and completed RunningCommand

The event may be observed only after the output worker's final stream cleanup.
``wait()`` remains the authority for process cleanup and error translation even
when ``is_alive`` has already recorded the exit code.  This separation preserves
the existing synchronous/background lifecycle while giving await a precise
non-blocking notification seam.

Verification and Implementation Sequence
========================================

``tests/sh_test.py::AsyncAwaitContractTests`` is the behavioral contract locus.
The exact constructed-instance assertion uses ``Command.RunningCommandCls``,
the repository's existing substitution seam.  The pause/final-write case
observes the event-loop boundary; completed properties observe ``OProc`` state;
the text case observes retained decoding policy; and the repeated-await case
observes identity, PID, bytes, exit code, and an invocation side effect.
``tests/shawait_verification_map.json`` remains the bidirectional canonical
requirement-to-verification index.

Implementation ordering is:

#. Replace the contract placeholders with the focused asynchronous behavior
   cases while retaining their names and verification map.
#. Change only ``RunningCommand.__await__`` to call the established finalization
   boundary and select ``self`` or decoded stdout from ``call_args``.
#. Run the five focused requirements plus map integrity, then the existing
   neighboring async success and error tests.

This order exposes regressions in completion timing, exception translation,
decoding, identity, and respawn behavior without changing production behavior
during this architecture phase.

Requirement-to-Architecture Map
===============================

.. list-table:: Canonical traceability
   :header-rows: 1

   * - Requirement and verification
     - Procedures
     - Owning loci and test seam
   * - SHAWAIT-001;
       ``test_shawait_001_return_cmd_await_returns_constructed_running_command``
     - ``construct_async_invocation``;
       ``await_completed_async_result``
     - ``Command.__call__`` / ``Command.RunningCommandCls`` construction;
       ``RunningCommand.__await__`` identity selection
   * - SHAWAIT-002;
       ``test_shawait_002_await_stays_pending_until_process_and_output_complete``
     - ``publish_async_output_completion``;
       ``await_completed_async_result``
     - ``output_thread`` completion publication;
       ``OProc.wait`` cleanup; ``RunningCommand.__await__`` event consumption
   * - SHAWAIT-003;
       ``test_shawait_003_completed_command_exposes_stdout_stderr_and_exit_code``
     - ``publish_async_output_completion``;
       ``await_completed_async_result``
     - ``OProc`` captured bytes and exit state;
       ``RunningCommand`` public properties
   * - SHAWAIT-004;
       ``test_shawait_004_default_await_returns_fully_decoded_text_output``
     - ``construct_async_invocation``;
       ``await_completed_async_result``
     - ``Command.__call__`` retained invocation policy;
       ``RunningCommand.__await__`` decoding branch
   * - SHAWAIT-015;
       ``test_shawait_015_repeated_await_returns_same_command_without_respawn``
     - ``construct_async_invocation``;
       ``await_completed_async_result``
     - One ``RunningCommand`` / ``OProc`` construction;
       set ``asyncio.Event`` and idempotent ``RunningCommand.wait``
