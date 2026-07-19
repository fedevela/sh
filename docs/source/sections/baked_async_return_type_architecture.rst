:orphan:

Baked Async Return-Type Architecture
####################################

Status and Scope
================

This record assigns SHAWAIT-005 through SHAWAIT-008 and the procedures
``compose_command_baked_return_policy``,
``compose_module_baked_return_policy``, ``construct_async_invocation``, and
``await_completed_async_result`` to the existing single-module runtime.  It is
a companion to ``completed_async_command_result_architecture.rst`` and changes
no production behavior in this architecture phase.

The architectural delta is deliberately confined to existing copy-on-bake and
invocation boundaries.  It introduces no new public type, persistence owner,
thread, event, package, configuration, or deployment seam.

Architectural Decision
======================

The final ``call_args`` mapping created by ``Command.__call__`` is the sole
invocation-level authority for awaited return-type selection.  Command and
module bakes own isolated defaults, not results.  They feed their defaults into
the established special-keyword precedence chain::

   Command-class defaults
       < command/module baked defaults
       < invocation special arguments
       -> RunningCommand.call_args["return_cmd"]
       -> RunningCommand.__await__ result selection after completion

Each ``<`` means that the value on the right wins when both loci supply it.
Later bakes derive a new owner with the latest explicit value; they do not
mutate their source.  An invocation override is copied into only that
invocation contract.  ``RunningCommand.__await__`` must not inspect a
``Command``, ``Environment``, or ``SelfWrapper`` after construction because
those objects are defaults providers rather than owners of running state.

Placement and Ownership
=======================

Command special-argument contract
---------------------------------

**Requirements:** SHAWAIT-005, SHAWAIT-007, SHAWAIT-008.

**Procedures:** ``compose_command_baked_return_policy`` and
``construct_async_invocation``.

``Command._call_args`` owns the canonical key and process-wide fallback value
for ``return_cmd``.  ``Command._extract_call_args`` owns recognition of the
public ``_return_cmd`` spelling, removal from ordinary executable arguments,
normalization to the internal ``return_cmd`` spelling, and validation with the
other special arguments.  This existing contract must be reused at every bake
and invocation boundary; a separate async-only parser would create divergent
precedence and validation rules.

Command-level copy-on-bake boundary
-----------------------------------

**Requirements:** SHAWAIT-005, SHAWAIT-007, SHAWAIT-008.

**Verification:**
``test_shawait_005_command_baked_return_cmd_true_await_returns_completed_running_command``,
``test_shawait_007_later_command_bake_overrides_return_cmd_and_rebake_reverses_awaited_result_type``,
``test_shawait_008_command_baked_true_invocation_false_await_returns_text``, and
``test_shawait_008_command_baked_false_invocation_true_await_returns_completed_running_command_without_mutating_default``.

**Procedure:** ``compose_command_baked_return_policy``.

``Command.bake`` owns command-local composition.  It creates a distinct
``Command`` of the same environment-local class, copies the source
``_partial_call_args``, then overlays newly extracted special arguments.  The
new mapping therefore owns the later-bake value while the source command keeps
its earlier value.  ``_partial_baked_args`` continues to own ordinary argument
order independently of the special-keyword mapping.

The incoming contract is ``(*args, **kwargs)`` with underscore-prefixed public
special arguments.  The outgoing contract is a derived ``Command`` with
composed ``_partial_call_args`` and ``_partial_baked_args``.  Failed extraction,
validation, construction, or ordinary argument compilation returns no derived
command and leaves the source unchanged under normal Python assignment
semantics.

Module-level copy-on-bake and resolution boundary
-------------------------------------------------

**Requirements:** SHAWAIT-006, SHAWAIT-007, SHAWAIT-008.

**Verification:**
``test_shawait_006_module_baked_return_cmd_true_await_returns_completed_running_command_without_mutating_original_environment``,
``test_shawait_007_later_module_bake_overrides_return_cmd_and_rebake_reverses_awaited_result_type``,
``test_shawait_008_module_baked_true_invocation_false_await_returns_text``, and
``test_shawait_008_module_baked_false_invocation_true_await_returns_completed_running_command_without_mutating_default``.

**Procedure:** ``compose_module_baked_return_policy``.

``SelfWrapper.bake`` owns module-environment composition.  It copies
``Environment.baked_args``, overlays the later bake, and constructs a distinct
``SelfWrapper``.  ``SelfWrapper.__init__`` validates recognized defaults and
creates an environment-local ``Command`` subclass whose copied ``_call_args``
contains them.  It also creates a distinct ``Environment`` retaining the
publicly spelled baked arguments for later command resolution.

``Environment.__getitem__`` owns dynamic name lookup.  For executable names it
delegates to ``resolve_command`` with its environment-local ``Command`` class
and composed ``baked_args``.  ``resolve_command`` constructs that class and
uses ``Command.bake`` to place the environment defaults on the resolved
command.  The class-default and command-partial representations are compatible
parts of the existing module-bake design; their equal environment value is
resolved once more by the ordinary invocation overlay.  Neither representation
may be read at await time.

The original wrapper, its ``Environment``, the process-global ``Command``
class, and commands already resolved from any wrapper remain independent.
Later module bakes affect only command lookups through the newly returned
wrapper.  ``CommandNotFound`` remains owned by resolution, before any
``RunningCommand`` exists.

Invocation and awaited-result boundary
--------------------------------------

**Requirements:** SHAWAIT-005, SHAWAIT-006, SHAWAIT-007, SHAWAIT-008.

**Verification:** all eight focused obligations listed in the traceability
table below.

**Procedures:** ``construct_async_invocation`` and
``await_completed_async_result``.

``Command.__call__`` owns the final precedence merge.  It copies the owning
command class's ``_call_args``, overlays the command's ``_partial_call_args``,
then overlays invocation special arguments extracted from a copied keyword
mapping.  It passes that resolved mapping by reference to exactly one
``RunningCommand``.  Because ``async=True`` suppresses construction-time
waiting, both return policies initially expose that same awaitable object.

``RunningCommand`` owns the resulting invocation contract and process identity.
Its ``__await__`` method first consumes the existing output-completion event,
then delegates error and timeout classification to ``wait()``, and only then
reads its own ``call_args["return_cmd"]``.  True returns the completed
``RunningCommand`` itself; false returns decoded stdout through the existing
``str(self)`` contract.  No bake owner participates after construction.

Boundaries and Contracts
========================

The cooperating contracts are:

#. Public ``_return_cmd`` enters ``Command._extract_call_args`` and becomes the
   internal ``return_cmd`` key.  Existing special-argument validation and
   exception behavior are preserved.
#. ``Command.bake`` exposes an isolated derived-command contract through
   ``_partial_call_args``; latest explicit values replace earlier values while
   ordinary baked arguments retain their append order.
#. ``SelfWrapper.bake`` exposes an isolated derived-environment contract through
   ``Environment.baked_args`` and an environment-local ``Command`` class.
#. ``Environment.__getitem__`` and ``resolve_command`` adapt module defaults to
   the ordinary ``Command.bake`` contract when a command is resolved.
#. ``Command.__call__`` produces the authoritative per-invocation ``call_args``
   contract with invocation values last.
#. ``RunningCommand.__await__`` consumes only that invocation contract after
   the existing asynchronous completion and synchronous finalization seams.

The contract carries a boolean policy, not a second result object and not an
event payload.  No schema, serialization, migration, authorization, or external
protocol is involved.

Dependencies and Integration
============================

Dependency flow is inward from the public facades toward the invocation owner::

   SelfWrapper.bake -> SelfWrapper.__init__ -> Environment
       -> Environment.__getitem__ -> resolve_command -> Command.bake
                                                      |
   direct Command.bake -------------------------------+
                                                      v
                         Command.__call__ -> RunningCommand -> OProc
                                                  |
                                    output-completion event
                                                  |
                                                  v
                                  RunningCommand.__await__

``SelfWrapper`` depends on ``Environment`` and the copied ``Command``
abstraction; ``Environment`` depends on ``resolve_command``; resolution depends
on ``Command.bake``.  ``Command.__call__`` depends on ``RunningCommandCls``, and
``RunningCommand`` depends on ``OProc`` for process state.  There is no reverse
dependency from process/output workers into bake ownership.  Their only reverse
notification remains the payload-free, thread-safe completion event described
in the companion architecture record.

State transitions are copy-on-write until invocation and invocation-local
afterward::

   source command -- bake --> derived command with independent defaults
   source wrapper -- bake --> derived wrapper/environment with independent defaults
   selected command -- call --> one resolved call_args / RunningCommand / OProc
   process and output complete --> await finalizes --> object or text result

The synchronous integration seams are special-argument extraction, copy and
overlay, command resolution, construction, and final ``wait()``.  The
asynchronous seam is unchanged: the output worker publishes completion to the
event loop and ``RunningCommand.__await__`` consumes it.  Timeout,
``ErrorReturnCode``, cancellation, stream cleanup, and retry/repeated-await
ownership remain with the loci established by
``completed_async_command_result_architecture.rst``; return policy does not
translate, suppress, retry, or compensate for those outcomes.

Verification and Implementation Sequence
========================================

``tests/sh_test.py::AsyncAwaitContractTests`` is the behavioral seam for all
four requirements.  Command cases observe ``Command.bake`` isolation and
precedence.  Module cases observe ``SelfWrapper``/``Environment`` isolation,
resolution, and precedence.  Opposite invocation values observe the last
overlay and a later unoverridden invocation observes non-mutation.
``tests/shawait_verification_map.json`` remains the bidirectional canonical
requirement-to-verification index.

Implementation should proceed from the owning convergence point outward:

#. Replace the eight named placeholders without renaming their map entries.
#. If focused behavior exposes a defect, preserve ``Command.bake`` and
   ``SelfWrapper.bake`` copy semantics and correct only transfer of the final
   resolved ``return_cmd`` into ``RunningCommand.call_args`` or its consumption
   by ``RunningCommand.__await__``.
#. Avoid an async-specific bake path, mutable environment sharing, or await-time
   lookup of defaults; each would create a second policy authority.
#. Validate the eight focused behavior cases together with verification-map
   completeness and neighboring asynchronous result contracts.

Requirement-to-Architecture Map
===============================

.. list-table:: Canonical traceability
   :header-rows: 1

   * - Requirement and verification obligation
     - Procedures
     - Architectural loci
   * - SHAWAIT-005;
       ``test_shawait_005_command_baked_return_cmd_true_await_returns_completed_running_command``
     - ``compose_command_baked_return_policy``;
       ``construct_async_invocation``; ``await_completed_async_result``
     - ``Command._extract_call_args`` and ``Command.bake`` own the baked
       default; ``Command.__call__`` owns resolution; ``RunningCommand.__await__``
       owns post-completion selection
   * - SHAWAIT-006;
       ``test_shawait_006_module_baked_return_cmd_true_await_returns_completed_running_command_without_mutating_original_environment``
     - ``compose_module_baked_return_policy``;
       ``construct_async_invocation``; ``await_completed_async_result``
     - ``SelfWrapper.bake`` / ``SelfWrapper.__init__`` own isolated module
       defaults; ``Environment.__getitem__`` / ``resolve_command`` adapt them;
       invocation and await loci resolve and consume them
   * - SHAWAIT-007;
       ``test_shawait_007_later_command_bake_overrides_return_cmd_and_rebake_reverses_awaited_result_type``
     - ``compose_command_baked_return_policy``;
       ``construct_async_invocation``; ``await_completed_async_result``
     - Successive ``Command.bake`` instances own reversible copy-on-bake
       values; the selected instance feeds the invocation contract
   * - SHAWAIT-007;
       ``test_shawait_007_later_module_bake_overrides_return_cmd_and_rebake_reverses_awaited_result_type``
     - ``compose_module_baked_return_policy``;
       ``construct_async_invocation``; ``await_completed_async_result``
     - Successive ``SelfWrapper.bake`` environments own reversible values;
       resolution feeds only the selected environment's value forward
   * - SHAWAIT-008;
       ``test_shawait_008_command_baked_true_invocation_false_await_returns_text``
     - ``compose_command_baked_return_policy``;
       ``construct_async_invocation``; ``await_completed_async_result``
     - ``Command.__call__`` overlays invocation false without mutating
       ``Command._partial_call_args``; await selects text
   * - SHAWAIT-008;
       ``test_shawait_008_module_baked_true_invocation_false_await_returns_text``
     - ``compose_module_baked_return_policy``;
       ``construct_async_invocation``; ``await_completed_async_result``
     - ``Command.__call__`` overlays invocation false without mutating module
       defaults; await selects text
   * - SHAWAIT-008;
       ``test_shawait_008_command_baked_false_invocation_true_await_returns_completed_running_command_without_mutating_default``
     - ``compose_command_baked_return_policy``;
       ``construct_async_invocation``; ``await_completed_async_result``
     - ``Command.__call__`` overlays invocation true only into its
       ``call_args``; await returns that completed ``RunningCommand``
   * - SHAWAIT-008;
       ``test_shawait_008_module_baked_false_invocation_true_await_returns_completed_running_command_without_mutating_default``
     - ``compose_module_baked_return_policy``;
       ``construct_async_invocation``; ``await_completed_async_result``
     - ``Command.__call__`` overlays invocation true only into its
       ``call_args``; await returns that completed ``RunningCommand``
