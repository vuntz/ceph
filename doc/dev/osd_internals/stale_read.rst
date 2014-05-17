Preventing Stale Reads
======================

We write synchronously to all replicas before sending an ack to the client,
which ensures that we do not introduce potential inconsistency in the write
path.  However, we only read from one replica, and we use whatever OSDMap we
happen to have.  In most cases, this is fine: our map either is correct, or
the OSD that we think is the primary for the object knows that it is not.

They key is to ensure that this is always true.  In particular, we need to
ensure that an OSD that is fenced off from its peers and has not learned about
a map update does not continue to service read requests from similarly stale
clients at any point after which a new primary may have been allowed to make
a write.

We accomplish this with the following rule: no OSD is allowed to
service a read unless it has seen a heartbeat from the current
pg_interval_t less that pg_pool_t::read_interval seconds ago.

The interval timer is implicitly renewed by the primary OSD for an
interval by the OSDPing heartbeats.

Before a new primary OSD in a subsequent interval is allowed to
service writes, it must have either:

 * be certain that all OSDs in the prior interval(s) know that the
   past interval has concluded (usually as a side-effect of probing
   them during peering), or
 * be certain that the prior interval's read_interval window has
   passed as the prior OSDs are no longer servicing reads.

The key piece of information that the new primary needs is an upper
bound on when that time period has completed.

Because we do not want to be concerned about time sync issues, we
express the end of that interval in terms of "readable seconds
remaining" whenever exchanging messages over the wire, and in terms of
a timestamp when represented locally.  (We assume that any clock
jitter on the local node is not significant.)

Primary
-------

* include send time in all heartbeats
* on ack, for that peer, note our send time ("acked hb start")

Replica, on heartbeat
---------------------

* note peer's start_time ("hb start")

On read
-------

* readable_until = MIN(peers acked hb starts) + read_interval
* defer read if now >= readable_until.

When sending pg_notify_t
------------------------

* include MAX(peers' hb starts) + read_interval - now (this is an
  upper bound on readable time remaining)

During peering
--------------

* note max readable_until value for all notifies (now + readable time remaining)
