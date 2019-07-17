# Scheduled Changes

_NOTE: This format comes from the
[Rust language RFC process](https://github.com/rust-lang/rfcs)._

- Feature or Prototype: scheduled_changes
- Start Date: 2019-07-17
- RFC PR: (leave this empty, populate once the PR is created with a github link to the PR)
- Issue: (once approved, this will link to the issue tracker for implementation of the RFC)

# Summary
[summary]: #summary

Allow users to set a scheduled date+time for a batch change.  The batch change will be processed close to the time specified.

# Motivation
[motivation]: #motivation

Often times, users need to "schedule" a DNS change that runs at a pre-determined date and time.  The typical use case is that personnel performing some kind of maintenance off hours that requires DNS changes, and the field technician does not have access to VinylDNS.

# Design and Goals
[design]: #design-and-goals

## Normal Flows

### Submission

1. User sets the scheduled time on a batch change 
1. User submits the batch change 
1. System validates that the batch change has no hard or soft errors.
    1. As the expected use is to run at any time with no personnel present, we do not want to accept batch changes with any errors
1. System saves the batch change in a "Pending" state.

### Processing

1. VinylDNS polls for scheduled changes on an interval (e.g. once per minute). The query to be used will check for any batch change that has a "Pending" state with a scheduled time that has expired (< NOW)
1. VinylDNS detects a change is "ready" for processing
1. VinylDNS submits the batch change id to the Message Queue for processing
1. VinylDNS receives the message with the batch change id from the Message Queue
1. VinylDNS loads the batch change from the database
1. VinylDNS re-runs validations to make sure the batch change is still in good order
    1. It is possible for DNS to shift while the batch change was waiting for processing.  We must make sure that the change is still able to be processed.  Question: what do we do if a scheduled change is no longer valid?
1. VinylDNS processes each change in the batch
1. Once all changes are complete, VinylDNS moves the batch change to a `Complete` status
    1. If some changes failed while others succeeded, the batch change will be marked as `PartialFailure`
    1. If all changes failed, the batch change will be marked as `Failed`

## Exception Flows

### Processing - Node Failure

The design uses a Message Queue with retries to handle the possibility of a node failure.  If a node dies, the message will be retried (for SQS this is up to 100 times) to help ensure that the message is implemented.  Due to the "retry" nature, we must make sure that batch change processing is _idempotent_.  The following steps ensure that batch change processing is idempotent.

1. Use the Message Queue to handle scheduled changes
1. When the batch change is loaded, do not attempt to re-process any single changes marked as `Complete` or `Failed`
1. Process all of the changes in the same process (i.e. do not submit individual record changes to a message queue for processing).  Create an FS2 queue for record changes and load all changes onto the queue.  The `CommandHandler` will stop polling the Message Queue for changes until the queue is exhausted.
    1. The alternative design would submit individual record set changes to the Message Queue for processing.  The major drawback to that design is what to do on node failure?  There is no way to know which messages are in-flight for processing, which ones are queued up, etc.  While record changes are idempotent, we open up the possibility of sending the same record change multiple times, processing it in parallel on separate nodes, introducing some odd race conditions.  Record changes are meant to be _idempotent_, not _multiple concurrent processing safe_.
    1. The drawback to this approach is that it will take longer to process record changes for larger batch changes.

### Processing - Validation Errors

Any validation errors, hard or soft errors, must advance the batch change to a `ManualReview` status.

1. When re-validating the batch change, if any errors happen (hard or soft), save the errors on the batch change and advance it to a `ManualReview` status
1. Raise a notification of `ScheduledChangeFailed` to notify someone to take action (e.g. notify an on-call engineer).

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?

# Outcome(s)
[outcome]: #outcome

Was this RFC implemented, subsumed by another RFC or implementation, in-waiting,
or discarded?

# References
[references]: #references
