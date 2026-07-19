:orphan:

Completed Async Command Result Logic
####################################

This artifact records the implementation procedure for preserving completed
asynchronous command results.  It is intentionally expressed independently of
Python syntax; the owning runtime loci are ``SelfWrapper.bake``,
``Environment.__getitem__``, ``resolve_command``, ``Command.bake``,
``Command.__call__``, ``RunningCommand.__await__``,
``RunningCommand.wait``, ``OProc.wait``, and ``output_thread`` in ``sh.py``.

Procedure: compose_command_baked_return_policy
================================================

.. code-block:: text

   PROCEDURE compose_command_baked_return_policy(source_command, bake_args)
     REQUIREMENT_IDS: SHAWAIT-005, SHAWAIT-007, SHAWAIT-008
     VERIFICATION:
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_005_command_baked_return_cmd_true_await_returns_completed_running_command
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_007_later_command_bake_overrides_return_cmd_and_rebake_reverses_awaited_result_type
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_008_command_baked_true_invocation_false_await_returns_text
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_008_command_baked_false_invocation_true_await_returns_completed_running_command_without_mutating_default

     PRECONDITIONS
       source_command is an executable Command, possibly produced by an earlier bake
       bake_args may contain _return_cmd and ordinary command arguments

     VALIDATE / NORMALIZE
       copy bake_args before extracting special arguments
       extract each recognized underscore-prefixed special argument and remove it
         from the ordinary arguments
       validate the extracted special arguments against Command._call_args
       IF extraction, validation, or ordinary argument compilation fails
         propagate the existing exception
         leave source_command unchanged and return no derived Command
       END IF

     COMPOSE
       construct a distinct derived Command for source_command._path
       copy source_command._partial_call_args into derived._partial_call_args
       update derived._partial_call_args with the newly extracted special arguments
       IF both mappings contain return_cmd
         the new bake's return_cmd replaces the earlier baked value
       END IF
       append source_command._partial_baked_args, then newly compiled ordinary
         arguments, preserving normal positional bake order

     TRANSITION / RETURN
       SOURCE_COMMAND -- successful bake --> DERIVED_COMMAND
       retain source_command and every previously derived Command unchanged
       return DERIVED_COMMAND with its independent composed defaults

     REPEATED BAKE
       apply this procedure to the most recently derived Command
       each successful later bake creates another Command and the latest explicit
         return_cmd becomes that Command's baked default
       rebaking return_cmd to an earlier value reverses only subsequent invocations
         through the newly returned Command
   END PROCEDURE

Procedure: compose_module_baked_return_policy
===============================================

.. code-block:: text

   PROCEDURE compose_module_baked_return_policy(source_environment, bake_kwargs)
     REQUIREMENT_IDS: SHAWAIT-006, SHAWAIT-007, SHAWAIT-008
     VERIFICATION:
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_006_module_baked_return_cmd_true_await_returns_completed_running_command_without_mutating_original_environment
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_007_later_module_bake_overrides_return_cmd_and_rebake_reverses_awaited_result_type
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_008_module_baked_true_invocation_false_await_returns_text
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_008_module_baked_false_invocation_true_await_returns_completed_running_command_without_mutating_default

     PRECONDITIONS
       source_environment is the original sh wrapper or a previously baked wrapper
       bake_kwargs contains only candidate environment-wide command defaults

     COMPOSE / VALIDATE
       copy source_environment.baked_args into composed_baked_args
       update composed_baked_args with bake_kwargs
       IF both mappings contain _return_cmd
         the new bake's _return_cmd replaces the earlier environment value
       END IF
       extract and validate recognized special arguments through Command
       IF extraction or validation fails
         propagate the existing exception
         leave source_environment unchanged and return no derived environment
       END IF

     ISOLATE / TRANSITION
       copy the base Command class metadata into an environment-local Command class
       copy base Command._call_args and update the copy with composed special defaults
       construct a distinct Environment retaining composed_baked_args
       SOURCE_ENVIRONMENT -- successful bake --> DERIVED_ENVIRONMENT
       do not mutate source_environment, the process-global Command defaults, or
         commands already resolved from either environment

     RESOLVE COMMAND
       WHEN DERIVED_ENVIRONMENT receives a command-name lookup
         resolve the executable path
         IF no executable or builtin is resolvable
           raise the existing CommandNotFound outcome
         END IF
         construct the environment-local Command
         bake composed_baked_args into the resolved Command
         return that Command with the environment's final return_cmd default
       END WHEN

     REPEATED BAKE
       apply this procedure to the most recently derived Environment
       latest explicit _return_cmd wins for commands resolved afterward
       rebaking _return_cmd to an earlier value creates another isolated Environment
         and reverses only later resolutions through that returned Environment
   END PROCEDURE

Procedure: construct_async_invocation
=====================================

.. code-block:: text

   PROCEDURE construct_async_invocation(command, positional_args, keyword_args)
     REQUIREMENT_IDS: SHAWAIT-001, SHAWAIT-004, SHAWAIT-005, SHAWAIT-006,
                      SHAWAIT-007, SHAWAIT-008, SHAWAIT-015
     VERIFICATION:
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_001_return_cmd_await_returns_constructed_running_command
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_004_default_await_returns_fully_decoded_text_output
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_005_command_baked_return_cmd_true_await_returns_completed_running_command
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_006_module_baked_return_cmd_true_await_returns_completed_running_command_without_mutating_original_environment
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_007_later_command_bake_overrides_return_cmd_and_rebake_reverses_awaited_result_type
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_007_later_module_bake_overrides_return_cmd_and_rebake_reverses_awaited_result_type
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_008_command_baked_true_invocation_false_await_returns_text
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_008_module_baked_true_invocation_false_await_returns_text
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_008_command_baked_false_invocation_true_await_returns_completed_running_command_without_mutating_default
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_008_module_baked_false_invocation_true_await_returns_completed_running_command_without_mutating_default
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_015_repeated_await_returns_same_command_without_respawn

     PRECONDITIONS
       command resolves to an executable Command
       keyword_args contains _async=True
       invocation occurs while an asyncio event loop is running

     LOAD / RECEIVE
       begin with the command class's default special arguments
       overlay command._partial_call_args, containing the final command-level or
         module-level baked values
       extract invocation special arguments from keyword_args and overlay them last
       therefore invocation return_cmd overrides the baked default for this call only
       preserve the final resolved async, return_cmd, encoding, decode_errors,
         ok_code, timeout, stdout, and stderr settings in call_args
       compile the executable path and ordinary arguments into cmd

     VALIDATE
       apply the existing special-argument validators before process creation
       IF validation or argument compilation fails
         propagate the existing exception
         create no RunningCommand and launch no subprocess
       END IF

     TRANSITION / DELEGATE
       CREATED -- async=True --> NONBLOCKING_INVOCATION
       construct exactly one RunningCommand through Command.RunningCommandCls
         using cmd, call_args, stdin, stdout, and stderr
       within that RunningCommand, create one unset aio_output_complete event
         bound to the running event loop
       because async=True, do not synchronously wait in RunningCommand.__init__
       spawn exactly one OProc and retain it on that RunningCommand
       NONBLOCKING_INVOCATION -- OProc spawned --> RUNNING

     RETURN
       return the one constructed RunningCommand regardless of return_cmd
       retain return_cmd on that same object's call_args for await-time selection
       do not write the invocation override back to command._partial_call_args,
         its environment, its class defaults, or any sibling Command

     REPEATED INVOCATION RULE
       awaiting the returned object never re-enters Command.__call__,
         RunningCommand.__init__, or OProc construction
   END PROCEDURE

Procedure: publish_async_output_completion
==========================================

.. code-block:: text

   PROCEDURE publish_async_output_completion(running_command, output_streams,
                                              process, event_loop)
     REQUIREMENT_IDS: SHAWAIT-002, SHAWAIT-003
     VERIFICATION:
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_002_await_stays_pending_until_process_and_output_complete
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_003_completed_command_exposes_stdout_stderr_and_exit_code

     PRECONDITIONS
       running_command is the object created for this invocation
       running_command.aio_output_complete is unset
       output_streams contains each internally managed stdout or stderr reader

     LOAD / RECEIVE
       register every managed output stream with the output poller

     ITERATE
       WHILE any registered stream can still produce data
         poll for readable, hangup, or error events
         FOR EACH readable or hung-up stream
           read its available bytes through its StreamReader
           aggregate retained stdout bytes into process._stdout
           aggregate retained stderr bytes into process._stderr
           forward configured callbacks, destinations, or pipe queues
           IF the StreamReader reports permanent completion
             unregister that stream
           END IF
         END FOR
         IF the established timeout or stop-output guard is set
           leave the polling loop under the existing cleanup policy
         END IF
       END WHILE

     ORDER / TRANSITION
       do not publish completion merely because an early write was read
       do not publish completion while a paused subprocess remains alive
       after stream polling ends, inspect process liveness under the wait lock
       WHILE process remains alive
         wait for a liveness change without blocking the event loop thread
       END WHILE
       record the adjusted actual exit_code when process termination is observed
       RUNNING -- subprocess terminated --> EXIT_OBSERVED
       close stdout and stderr readers so buffered final bytes are flushed
       EXIT_OBSERVED -- both stream readers closed --> OUTPUT_COMPLETE

     EMIT
       schedule aio_output_complete.set on event_loop with its thread-safe bridge
       publish no result before OUTPUT_COMPLETE

     ON TIMEOUT
       apply the invocation's timeout signal through the existing OProc policy
       continue termination and stream cleanup before publishing completion

     ON OUTPUT THREAD FAILURE
       preserve the existing thread-exception recording behavior
       do not fabricate stdout, stderr, exit_code, or a replacement command
   END PROCEDURE

Procedure: await_completed_async_result
=======================================

.. code-block:: text

   PROCEDURE await_completed_async_result(running_command)
     REQUIREMENT_IDS: SHAWAIT-001, SHAWAIT-002, SHAWAIT-003, SHAWAIT-004,
                      SHAWAIT-005, SHAWAIT-006, SHAWAIT-007, SHAWAIT-008,
                      SHAWAIT-012, SHAWAIT-015
     VERIFICATION:
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_001_return_cmd_await_returns_constructed_running_command
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_002_await_stays_pending_until_process_and_output_complete
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_003_completed_command_exposes_stdout_stderr_and_exit_code
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_004_default_await_returns_fully_decoded_text_output
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_005_command_baked_return_cmd_true_await_returns_completed_running_command
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_006_module_baked_return_cmd_true_await_returns_completed_running_command_without_mutating_original_environment
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_007_later_command_bake_overrides_return_cmd_and_rebake_reverses_awaited_result_type
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_007_later_module_bake_overrides_return_cmd_and_rebake_reverses_awaited_result_type
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_008_command_baked_true_invocation_false_await_returns_text
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_008_module_baked_true_invocation_false_await_returns_text
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_008_command_baked_false_invocation_true_await_returns_completed_running_command_without_mutating_default
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_008_module_baked_false_invocation_true_await_returns_completed_running_command_without_mutating_default
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_012_text_await_yields_to_sentinel_before_returning_output
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_012_return_cmd_await_yields_to_sentinel_before_returning_completed_command
       tests/sh_test.py::AsyncAwaitContractTests::test_shawait_015_repeated_await_returns_same_command_without_respawn

     PRECONDITIONS
       running_command is the invocation's original RunningCommand
       running_command.call_args contains the resolved return_cmd value
       running_command.aio_output_complete is the asyncio.Event created on the
         event loop that initiated this asynchronous command

     WAIT / SUSPEND
       select no result branch from return_cmd before completion
       IF the event is unset
         transition AWAIT_STARTED -- event unset --> AWAIT_SUSPENDED
         await running_command.aio_output_complete.wait through asyncio
         yield the event-loop thread so every ready task, including a concurrently
           scheduled short-delay sentinel, can make progress while the command runs
         perform no blocking process wait, thread join, output decode, or
           RunningCommand return while AWAIT_SUSPENDED
         remain suspended until the output worker schedules event.set on the
           owning event loop after process and output completion
         transition AWAIT_SUSPENDED -- event set --> COMPLETION_SIGNAL_RECEIVED
       ELSE
         transition AWAIT_STARTED -- event already set --> COMPLETION_SIGNAL_RECEIVED
         continue immediately because asyncio.Event remains set for repeated awaits
       END IF

     FINALIZE
       delegate to running_command.wait
       running_command.wait delegates to OProc.wait under the process wait lock
       IF the process was not already reaped
         reap it and store its adjusted actual exit_code
       END IF
       join input, output, and background threads under existing cleanup ordering
       set _waited_until_completion before applying the established completion
         classification, so subsequent wait calls do not repeat process cleanup
       validate timeout and acceptable exit-code policy
       IF timeout policy classifies completion as timed out
         raise the existing TimeoutException with no return value
       ELSE IF exit_code is not accepted by ok_code and piping policy
         raise the existing ErrorReturnCode subclass containing captured streams
       END IF
       OUTPUT_COMPLETE -- successful finalization --> COMPLETED_SUCCESS

     DECIDE
       IF running_command.call_args[return_cmd] is True
         result := running_command
       ELSE
         result := decode running_command.process.stdout using
           running_command.call_args[encoding] and
           running_command.call_args[decode_errors]
       END IF

     RETURN
       return no result before COMPLETION_SIGNAL_RECEIVED and successful finalization
       return result

     REPEATED AWAIT
       reuse the already-set aio_output_complete event
       reuse the idempotently finalized RunningCommand and its existing OProc
       do not construct or launch another command or process
       IF return_cmd is True
         return the identical running_command object, preserving identity, pid,
           stdout bytes, stderr bytes, and exit_code
       ELSE
         decode the same completed stdout bytes with the same invocation settings
       END IF

     ON AWAIT CANCELLATION
       propagate cancellation to the awaiting task
       do not replace or respawn running_command; its process lifecycle continues
         under the established OProc threads and a later await observes the same event
   END PROCEDURE

Traceability Matrix
===================

.. list-table:: Requirement and verification ownership
   :header-rows: 1

   * - Requirement
     - Verification obligation
     - Complete procedure
   * - SHAWAIT-001
     - ``test_shawait_001_return_cmd_await_returns_constructed_running_command``
     - ``construct_async_invocation``; ``await_completed_async_result``
   * - SHAWAIT-002
     - ``test_shawait_002_await_stays_pending_until_process_and_output_complete``
     - ``publish_async_output_completion``; ``await_completed_async_result``
   * - SHAWAIT-003
     - ``test_shawait_003_completed_command_exposes_stdout_stderr_and_exit_code``
     - ``publish_async_output_completion``; ``await_completed_async_result``
   * - SHAWAIT-004
     - ``test_shawait_004_default_await_returns_fully_decoded_text_output``
     - ``construct_async_invocation``; ``await_completed_async_result``
   * - SHAWAIT-005
     - ``test_shawait_005_command_baked_return_cmd_true_await_returns_completed_running_command``
     - ``compose_command_baked_return_policy``; ``construct_async_invocation``;
       ``await_completed_async_result``
   * - SHAWAIT-006
     - ``test_shawait_006_module_baked_return_cmd_true_await_returns_completed_running_command_without_mutating_original_environment``
     - ``compose_module_baked_return_policy``; ``construct_async_invocation``;
       ``await_completed_async_result``
   * - SHAWAIT-007
     - ``test_shawait_007_later_command_bake_overrides_return_cmd_and_rebake_reverses_awaited_result_type``
     - ``compose_command_baked_return_policy``; ``construct_async_invocation``;
       ``await_completed_async_result``
   * - SHAWAIT-007
     - ``test_shawait_007_later_module_bake_overrides_return_cmd_and_rebake_reverses_awaited_result_type``
     - ``compose_module_baked_return_policy``; ``construct_async_invocation``;
       ``await_completed_async_result``
   * - SHAWAIT-008
     - ``test_shawait_008_command_baked_true_invocation_false_await_returns_text``
     - ``compose_command_baked_return_policy``; ``construct_async_invocation``;
       ``await_completed_async_result``
   * - SHAWAIT-008
     - ``test_shawait_008_module_baked_true_invocation_false_await_returns_text``
     - ``compose_module_baked_return_policy``; ``construct_async_invocation``;
       ``await_completed_async_result``
   * - SHAWAIT-008
     - ``test_shawait_008_command_baked_false_invocation_true_await_returns_completed_running_command_without_mutating_default``
     - ``compose_command_baked_return_policy``; ``construct_async_invocation``;
       ``await_completed_async_result``
   * - SHAWAIT-008
     - ``test_shawait_008_module_baked_false_invocation_true_await_returns_completed_running_command_without_mutating_default``
     - ``compose_module_baked_return_policy``; ``construct_async_invocation``;
       ``await_completed_async_result``
   * - SHAWAIT-012
     - ``test_shawait_012_text_await_yields_to_sentinel_before_returning_output``
     - ``await_completed_async_result``
   * - SHAWAIT-012
     - ``test_shawait_012_return_cmd_await_yields_to_sentinel_before_returning_completed_command``
     - ``await_completed_async_result``
   * - SHAWAIT-015
     - ``test_shawait_015_repeated_await_returns_same_command_without_respawn``
     - ``construct_async_invocation``; ``await_completed_async_result``
