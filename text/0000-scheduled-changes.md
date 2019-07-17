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
    1. It is possible for DNS to shift while the batch change was waiting for processing.  We must make sure that the change is still able to be processed

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
