=========================================
Raft Consensus Algorithm in ScyllaDB
=========================================

Introduction
--------------
ScyllaDB was originally designed, following Apache Cassandra, to use gossip for topology and schema updates and the Paxos consensus algorithm for
strong data consistency (:doc:`LWT </using-scylla/lwt>`). To achieve stronger consistency without performance penalty, ScyllaDB has turned to Raft - a consensus algorithm designed as an alternative to both gossip and Paxos.

Raft is a consensus algorithm that implements a distributed, consistent, replicated log across members (nodes). Raft implements consensus by first electing a distinguished leader, then giving the leader complete responsibility for managing the replicated log. The leader accepts log entries from clients, replicates them on other servers, and tells servers when it is safe to apply log entries to their state machines.

Raft uses a heartbeat mechanism to trigger a leader election. All servers start as followers and remain in the follower state as long as they receive valid RPCs (heartbeat) from a leader or candidate. A leader sends periodic heartbeats to all followers to maintain his authority (leadership). Suppose a follower receives no communication over a period called the election timeout. In that case, it assumes no viable leader and begins an election to choose a new leader.

Leader selection is described in detail in the `Raft paper <https://raft.github.io/raft.pdf>`_.

ScyllaDB can use Raft to maintain schema updates in every node (see below). Any schema update, like ALTER, CREATE or DROP TABLE, is first committed as an entry in the replicated Raft log, and, once stored on most replicas, applied to all nodes **in the same order**, even in the face of a node or network failures.

Upcoming ScyllaDB releases will use Raft to guarantee consistent topology updates similarly.

.. _raft-quorum-requirement:

Quorum Requirement
-------------------

Raft requires at least a quorum of nodes in a cluster to be available. If multiple nodes fail
and the quorum is lost, the cluster is unavailable for schema updates. See :ref:`Handling Failures <raft-handling-failures>`
for information on how to handle failures.

Note that when you have a two-DC cluster with the same number of nodes in each DC, the cluster will lose the quorum if one
of the DCs is down.
**We recommend configuring three DCs per cluster to ensure that the cluster remains available and operational when one DC is down.**

.. _enabling-raft-existing-cluster:

Enabling Raft
---------------

.. note::
  In ScyllaDB 5.2 and ScyllaDB Enterprise 2023.1 Raft is Generally Available and can be safely used for consistent schema management.
  In further versions, it will be mandatory.

ScyllaDB Open Source 5.2 and later, and ScyllaDB Enterprise 2023.1 and later come equipped with a procedure that can setup Raft-based consistent cluster management in an existing cluster. We refer to this as the **Raft upgrade procedure** (do not confuse with the :doc:`ScyllaDB version upgrade procedure </upgrade/index/>`).

.. warning::
    Once enabled, Raft cannot be disabled on your cluster. The cluster nodes will fail to restart if you remove the Raft feature.

To enable Raft in an existing cluster, you need to enable the ``consistent_cluster_management`` option in the ``scylla.yaml`` file 
for **each node** in the cluster: 

#. Ensure that the schema is synchronized in the cluster by executing :doc:`nodetool describecluster </operating-scylla/nodetool-commands/describecluster>` on each node and ensuring that the schema version is the same on all nodes.
#. Perform a :doc:`rolling restart </operating-scylla/procedures/config-change/rolling-restart/>`, updating the ``scylla.yaml`` file for **each node** in the cluster before restarting it to enable the ``consistent_cluster_management`` option:

    .. code-block:: yaml

       consistent_cluster_management: true

When all the nodes in the cluster and updated and restarted, the cluster will start the **Raft upgrade procedure**.
**You must then verify** that the Raft upgrade procedure has finished successfully. Refer to the :ref:`next section <verify-raft-procedure>`.

Alternatively, you can enable the ``consistent_cluster_management`` option when you are:

* Performing a rolling upgrade from version 5.1 to 5.2 or version 2022.x to 2023.1 by updating ``scylla.yaml`` before restarting each node. The Raft upgrade procedure will start as soon as the last node was upgraded and restarted. As above, this requires :ref:`verifying <verify-raft-procedure>` that the procedure successfully finishes.
* Creating a new cluster. This does not use the Raft upgrade procedure; instead, Raft is functioning in the cluster and managing schema right from the start.

Until all nodes are restarted with ``consistent_cluster_management: true``, it is still possible to turn this option back off. Once enabled on every node, it must remain turned on (or the node will refuse to restart).

.. _verify-raft-procedure:

Verifying that the Raft upgrade procedure finished successfully
========================================================================

The Raft upgrade procedure starts as soon as every node in the cluster restarts with ``consistent_cluster_management`` flag enabled in ``scylla.yaml``.

.. TODO: update the above sentence once 5.3 and later are released.

The procedure requires **full cluster availability** to correctly setup the Raft algorithm; after the setup finishes, Raft can proceed with only a majority of nodes, but this initial setup is an exception.
An unlucky event, such as a hardware failure, may cause one of your nodes to fail. If this happens before the Raft upgrade procedure finishes, the procedure will get stuck and your intervention will be required.

To verify that the procedure finishes, look at the log of every Scylla node (using ``journalctl _COMM=scylla``). Search for the following patterns:

* ``Starting internal upgrade-to-raft procedure`` denotes the start of the procedure,
* ``Raft upgrade finished`` denotes the end.

The following is an example of a log from a node which went through the procedure correctly. Some parts were truncated for brevity:

.. code-block:: console

    features - Feature SUPPORTS_RAFT_CLUSTER_MANAGEMENT is enabled
    raft_group0 - finish_setup_after_join: SUPPORTS_RAFT feature enabled. Starting internal upgrade-to-raft procedure.
    raft_group0_upgrade - starting in `use_pre_raft_procedures` state.
    raft_group0_upgrade - Waiting until everyone is ready to start upgrade...
    raft_group0_upgrade - Joining group 0...
    raft_group0 - server 624fa080-8c0e-4e3d-acf6-10af473639ca joined group 0 with group id 8f8a1870-5c4e-11ed-bb13-fe59693a23c9
    raft_group0_upgrade - Waiting until every peer has joined Raft group 0...
    raft_group0_upgrade - Every peer is a member of Raft group 0.
    raft_group0_upgrade - Waiting for schema to synchronize across all nodes in group 0...
    raft_group0_upgrade - synchronize_schema: my version: a37a3b1e-5251-3632-b6b4-a9468a279834
    raft_group0_upgrade - synchronize_schema: schema mismatches: {}. 3 nodes had a matching version.
    raft_group0_upgrade - synchronize_schema: finished.
    raft_group0_upgrade - Entering synchronize state.
    raft_group0_upgrade - Schema changes are disabled in synchronize state. If a failure makes us unable to proceed, manual recovery will be required.
    raft_group0_upgrade - Waiting for all peers to enter synchronize state...
    raft_group0_upgrade - All peers in synchronize state. Waiting for schema to synchronize...
    raft_group0_upgrade - synchronize_schema: collecting schema versions from group 0 members...
    raft_group0_upgrade - synchronize_schema: collected remote schema versions.
    raft_group0_upgrade - synchronize_schema: my version: a37a3b1e-5251-3632-b6b4-a9468a279834
    raft_group0_upgrade - synchronize_schema: schema mismatches: {}. 3 nodes had a matching version.
    raft_group0_upgrade - synchronize_schema: finished.
    raft_group0_upgrade - Schema synchronized.
    raft_group0_upgrade - Raft upgrade finished.

In a functioning cluster with good network connectivity the procedure should take no more than a few seconds.
Network issues may cause the procedure to take longer, but if all nodes are alive and the network is eventually functional (each pair of nodes is eventually connected), the procedure will eventually finish.

Note the following message, which appears in the log presented above:

.. code-block:: console

    Schema changes are disabled in synchronize state. If a failure makes us unable to proceed, manual recovery will be required.

During the procedure, there is a brief window while schema changes are disabled. This is when the schema change mechanism switches from the older unsafe algorithm to the safe Raft-based algorithm. If everything runs smoothly, this window will be unnoticeable; the procedure is designed to minimize that window's length. However, if the procedure gets stuck e.g. due to network connectivity problem, ScyllaDB will return the following error when trying to perform a schema change during this window:

.. code-block:: console

    Cannot perform schema or topology changes during this time; the cluster is currently upgrading to use Raft for schema operations.
    If this error keeps happening, check the logs of your nodes to learn the state of upgrade. The upgrade procedure may get stuck
    if there was a node failure.

In the next example, one of the nodes had a power outage before the procedure could finish. The following shows a part of another node's logs:

.. code-block:: console

    raft_group0_upgrade - Entering synchronize state.
    raft_group0_upgrade - Schema changes are disabled in synchronize state. If a failure makes us unable to proceed, manual recovery will be required.
    raft_group0_upgrade - Waiting for all peers to enter synchronize state...
    raft_group0_upgrade - wait_for_peers_to_enter_synchronize_state: node 127.90.69.3 not in synchronize state yet...
    raft_group0_upgrade - wait_for_peers_to_enter_synchronize_state: node 127.90.69.1 not in synchronize state yet...
    raft_group0_upgrade - wait_for_peers_to_enter_synchronize_state: retrying in a while...
    raft_group0_upgrade - wait_for_peers_to_enter_synchronize_state: node 127.90.69.1 not in synchronize state yet...
    raft_group0_upgrade - wait_for_peers_to_enter_synchronize_state: retrying in a while...
    ...
    raft_group0_upgrade - Raft upgrade procedure taking longer than expected. Please check if all nodes are live and the network is healthy. If the upgrade procedure does not progress even though the cluster is healthy, try performing a rolling restart of the cluster. If that doesn 't help or some nodes are dead and irrecoverable, manual recovery may be required. Consult the relevant documentation.
    raft_group0_upgrade - wait_for_peers_to_enter_synchronize_state: node 127.90.69.1 not in synchronize state yet...
    raft_group0_upgrade - wait_for_peers_to_enter_synchronize_state: retrying in a while...

.. TODO: the 'Consult the relevant documentation' message must be updated to point to this doc.

Note the following message:

.. code-block:: console

    raft_group0_upgrade - Raft upgrade procedure taking longer than expected. Please check if all nodes are live and the network is healthy. If the upgrade procedure does not progress even though the cluster is healthy, try performing a rolling restart of the cluster. If that doesn 't help or some nodes are dead and irrecoverable, manual recovery may be required. Consult the relevant documentation.

If the Raft upgrade procedure is stuck, this message will appear periodically in each node's logs.

The message suggests the initial course of action:

* Check if all nodes are alive.
* If a node is down but can be restarted, restart it.
* If all nodes are alive, ensure that the network is healthy: that every node is reachable from every other node.
* If all nodes are alive and the network is healthy, perform a :doc:`rolling restart </operating-scylla/procedures/config-change/rolling-restart/>` of the cluster.

One of the reasons why the procedure may get stuck is a pre-existing problem in schema definitions which causes schema to be unable to synchronize in the cluster. The procedure cannot proceed unless it ensures that schema is synchronized.
If **all nodes are alive and the network is healthy**, you performed a rolling restart, but the issue still persists, contact `ScyllaDB support <https://www.scylladb.com/product/support/>`_ for assistance.

If some nodes are **dead and irrecoverable**, you'll need to perform a manual recovery procedure. Consult :ref:`the section about Raft recovery <recovery-procedure>`.


Verifying that Raft is enabled
===============================

You can verify that Raft is enabled on your cluster by performing the following query on each node:

.. code-block:: sql

   cqlsh> SELECT * FROM system.scylla_local WHERE key = 'group0_upgrade_state';

The query should return:

   .. code-block:: console

     key                  | value
    ----------------------+--------------------------
     group0_upgrade_state | use_post_raft_procedures

    (1 rows)

on every node.

If the query returns 0 rows, or ``value`` is ``synchronize`` or ``use_pre_raft_procedures``, it means that the cluster is in the middle of the Raft upgrade procedure; consult the :ref:`relevant section <verify-raft-procedure>`.

If ``value`` is ``recovery``, it means that the cluster is in the middle of the manual recovery procedure. The procedure must be finished. Consult :ref:`the section about Raft recovery <recovery-procedure>`.

If ``value`` is anything else, it might mean data corruption or a mistake when performing the manual recovery procedure. The value will be treated as if it was equal to ``recovery`` when the node is restarted.

.. _raft-schema-changes:

Safe Schema Changes with Raft
-------------------------------
In ScyllaDB, schema is based on :doc:`Data Definition Language (DDL) </cql/ddl>`. In earlier ScyllaDB versions, schema changes were tracked via the gossip protocol, which might lead to schema conflicts if the updates are happening concurrently.

Implementing Raft eliminates schema conflicts and allows full automation of DDL changes under any conditions, as long as a quorum
of nodes in the cluster is available. The following examples illustrate how Raft provides the solution to problems with schema changes.

* A network partition may lead to a split-brain case, where each subset of nodes has a different version of the schema.

     With Raft, after a network split, the majority of the cluster can continue performing schema changes, while the minority needs to wait until it can rejoin the majority. Data manipulation statements on the minority can continue unaffected, provided the :ref:`quorum requirement <raft-quorum-requirement>` is satisfied.

* Two or more conflicting schema updates are happening at the same time. For example, two different columns with the same definition are simultaneously added to the cluster. There is no effective way to resolve the conflict - the cluster will employ the schema with the most recent timestamp, but changes related to the shadowed table will be lost.

     With Raft, concurrent schema changes are safe.



In summary, Raft makes schema changes safe, but it requires that a quorum of nodes in the cluster is available.


.. _raft-handling-failures:

Handling Failures
------------------

See :doc:`Handling Node Failures </troubleshooting/handling-node-failures>`.

.. _raft-learn-more:

Learn More About Raft
----------------------
* `The Raft Consensus Algorithm <https://raft.github.io/>`_
* `Achieving NoSQL Database Consistency with Raft in ScyllaDB <https://www.scylladb.com/tech-talk/achieving-nosql-database-consistency-with-raft-in-scylla/>`_ - A tech talk by Konstantin Osipov
* `Making Schema Changes Safe with Raft <https://www.scylladb.com/presentations/making-schema-changes-safe-with-raft/>`_ - A Scylla Summit talk by Konstantin Osipov (register for access)
* `The Future of Consensus in ScyllaDB 5.0 and Beyond <https://www.scylladb.com/presentations/the-future-of-consensus-in-scylladb-5-0-and-beyond/>`_ - A Scylla Summit talk by Tomasz Grabiec (register for access)

