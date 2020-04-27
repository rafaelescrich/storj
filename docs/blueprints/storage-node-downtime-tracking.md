# Storage Node Downtime Tracking

## Abstract

This document describes a means of tracking storage node downtime and using this information to suspend and disqualify.

## Background

The previous implementation of uptime reputation consisted of a ratio of successes to failures. The problem we encountered was that, due to the frequency of auditing any particular node being directly correlated with the number of pieces it holds, some nodes' reputations would quickly become destroyed over a relatively short period of downtime. To solve this problem we need a system which takes into account not only how many offline audits occur, but _when_ they occur as well.

## Design

The solution proposed here is to use a series of sliding windows to indicate a general timeframe in which offline audits occur. Each window will track whether a node was offline and/or online for any audits within its timeframe.
For example, we could have windows with a length of 24 hours, and if a node is both online and offline for any number of audits, both the online and offline fields of the window will be set to true. By compressing offline audits over the course of the window into a single value we can neutralize the damaging effect of frequent audits. We can use this information to suspend and disqualify nodes.

Storage node downtime can have a range of causes. For those storage node operators who may have fallen victim to a temporary issue, we want to give them a chance to diagnose and fix it before disqualifying them for good. For this reason, we are introducing suspension as a component of disqualification.

Suspension will be implemented by observing a set number of consecutive windows containing only offline audit results. If, after suspension is initiated, a node again receives the set number of consecutive offline-only windows, the node is disqualified.

### Database table

We will need a place to store these windows. A table in the satellite database will probably be sufficient.

Table
```sql
CREATE TABLE uptime_windows (
    NodeID BYTEA,
    Results BIT(2),
    WindowStart TIMESTAMP,
)
```
The `Results` column is a string of 2 bits where the least significant bit (big endian) corresponds to offline audits, and the most significant bit corresponds to online audits. The `WindowStart` column refers to the start boundary of the window.

API
```
type UptimeWindows interface {
    // Write uptime windows from a set of audit responses indicating whether a node was online or offline
    Write(ctx context.Context, nodeIDs map[storj.NodeID]bool, windowLength time.Duration) error
    // GetOffendingNodes returns nodes who have reached the max consecutive offline-only windows
    GetOffendingNodes(ctx context.Context, windowLength time.Duration, maxWindows int) (storj.NodeIDList, error)
    // Cleanup deletes entries which are no longer needed
    Cleanup(ctx context.Context, windowLength time.Duration, maxWindows int) error
}
```

Write
```sql
INSERT INTO uptime_windows VALUES (
    $1, $2, 
    CASE WHEN $3::bool IS TRUE THEN B'10'
        ELSE B'01'
    END
)
ON CONFLICT (NodeID, WindowStart)
DO UPDATE
SET Results = CASE WHEN $3::bool IS TRUE THEN Results || B'10'
        ELSE Results || B'01'
    END;
```

GetOffendingNodes
```sql
SELECT node_id 
FROM (
    SELECT node_id, COUNT(*) as n
    FROM uptime_windows
    WHERE results = B'01' AND WindowStart < $1
    GROUP BY node_id
) t
WHERE n >= $2;
```

Cleanup
```sql
DELETE FROM uptime_windows WHERE WindowStart < $1;
```

### A service for writing to and reading from windows

To write and read from these windows we need a couple of configurable values.
- WindowLength: length of time spanning a single uptime window
- MaxOfflineWindows: Number of consecutive windows with only offline audits before suspension or disqualification

These values need to live somewhere to be passed to the database. Maybe on a service for handling writes and reads on uptime windows.

```
type Service struct {
    windowLength      time.Duration
    maxOfflineWindows int

    uptimeWindowsDB   DB
    cache             overlay.DB
}
```

```
type Service interface {
    // Write uptime windows from a set of audit responses indicating whether a node was online or offline
    Write(ctx context.Context, nodeIDs map[storj.NodeID]bool) error
    // SuspendOrDisqualify suspends or disqualifies eligible storage nodes
    SuspendOrDisqualify(ctx context.Context) error
    // Cleanup deletes entries which are no longer needed
    Cleanup(ctx context.Context) error
}
```

#### Write
To write to the database we will give the audit reporter access to this service. The audit reporter receives the audit results of each node for a segment. We can use this information to build a map of nodes and their online/offline status and pass it to the UptimeWindows service to write to the database. By truncating the current time to the nearest multiple of the WindowLength on the service, the database can easily determine if any entries corresponding to the current window already exist and insert or update as needed.

#### SuspendOrDisqualify
When reading, what we want to know is whether any nodes have reached the maximum consecutive _complete_ offline windows, and if so, suspsend or disqualify them. To do this, we can call the uptime_windows database method, `GetOffendingNodes`, and pass in WindowLength and MaxOfflineWindows from the service. We look up the returned nodes in the nodes table to see if they are already suspended for having too many offline-only windows. If not, set the `uptime_suspended` column to the current time. If they are already suspended for uptime, we need to check to see if the number of required windows have elapsed since the time of suspension. If so, and if the windows all contain only offline results, disqualify the node.

NOTE: What about reinstating suspended nodes?
Since we only care about consecutive offline windows, a single online audit response would break the suspension. 
We can easily integrate this into the overlay cache method for updating audit reputation. If the audit response indicates the node was online, and it is currently suspended for uptime, erase the suspension.

#### Cleanup
In order to avoid the table becoming infinitely large, we need to delete entries after a certain point. We have two options here:
1) Since we only look for a certain number of consecutive windows, any entries beyond this amount are unnecessary and can be deleted.
2) Perhaps we want to keep old entries around longer than necessary in the event that a node wants to dispute their suspension or disqualification. In this case we simply need another configurable value to define how long we want to keep entries.

As mentioned above, we can give the audit reporter access to the service for writes, but when do we read and delete?

### UptimeWindows Chore

The job of the Chore is to manage calling the `Cleanup` and `SuspendOrDisqualify` methods of the UptimeWindows Service on an interval. We only need to check as often as a window is completed and, therefore, the interval should be set equal to the window length. On each iteration, we will first run `Cleanup` to delete unnecessary entries. Then we will run `SuspendOrDisqualify`.

## Rationale

### WIP

## Implementation

WIP

## Wrapup

WIP

## Open issues

Concerns with current design
- By setting the suspension/DQ criteria to only observe _consecutive_ offline-only windows, a node could dodge punishment by being online for just a single audit within the MaxOfflineWindows. For some nodes this could be a single audit out of several hundred. Disqualification based on audit failure could be extremely slow in this case. The size of this loophole depends on how long our windows are and how many consecutive offline windows we allow.

- Given that this design is concerned with windows containing _only_ offline results, this allows nodes with more data more chances to be online for an audit in any window. This runs counter to the notion that we should be stricter with nodes that, because they hold more data, have a higher potential negative impact on the network.

