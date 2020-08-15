# Adopt an RFC process

## Summary

[summary]: #summary

RFC PR: [casperlabs/rfcs#0001](https://github.com/casperlabs/rfcs/pull/0001)

Make specifications, ideas and implementation guidelines more developer friendly and accessible through the introduction of an easy to use RFCs process.

## Motivation

[motivation]: #motivation

Currently we are relying on a hodge-podge of tools for capturing specifications and ideas. Confluence is the place where documentation goes to die, while there is also the public [docs.casperlabs.io](https://docs.casperlabs.io), comments on Jira tickets, Google Docs (for better comments and versioning that Confluence), HackMD (for Markdown), Slack conversations, plain old text files on a developers machine and other things I have missed.

There is a need for a reasonably lightweight process to discuss and adopt ideas, not only on the tech side, but governance and economics as well. We want this process to serve those behind these ideas and those implementing them first, instead of being caught in project management tooling. If possible, it should also be friendly to outside contributors, setting a low barrier for them to contribute feedback or their own ideas.

Typical usecases are the discussion of refactoring or design decisions of the software, policies for governance, or schemes cooked up by the economics team.

By introducing a little bit of formality, we hope to keep these designs fun and stave off the introduction of more restrictive processes. Creating RFCs should be easy using Markdown and GitHub, tools that everyone is already familiar with.

We are largely based on the [Rust RFC process](https://github.com/rust-lang/rfcs), albeit simplified. This document will tell you all you need to know about creating an RFC.

## How to create an RFC

[guide-level-explanation]: #guide-level-explanation

Any RFC (short for "request for comments") starts out with an idea. Some ideas are small enough to be exhaustively discussed in a short Slack conversation before making it into a pull request, there is no need to create an RFC for these. However, if the discussions grows in size and requires feedback from multiple people, this is the process:

1. Fork the RFC repo at [casperlabs/rfcs](https://github.com/casperlabs/rfcs).
2. Create a new branch for your RFC on your private repo, name it accordingly, e.g. `my-new-proposal`.
3. Copy the `0000-template.md` from the root to `text/0000-my-new-proposal.md`.
4. Edit the file, creating the first draft of the RFC.
5. Create a pull request to the RFC repo at [casperlabs/rfcs](https://github.com/casperlabs/rfcs). This PR will have a number, which is the official RFC number.
6. Add one commit immediately that updates the file name and links inside the RFC with the assigned number. Afterwards, add a "Rendered" link pointing to the branch-latest file via GitHub on your RFCs branch for easier reading (e.g. `https://github.com/yourgitusername/rfcs/blob/my-new-proposal/text/1234-my-new-proposal.md`)

### Discussion and merging

The RFC now enters discussion period. Invite people to review it by requesting reviews and/or messaging them directly to get them to comment on it. This will likely result in proposed changes, which should be addressed through additional commits.

Please remind people to keep feedback and comments to the PR on GitHub if possible, as it is otherwise likely that the author will have to repeat himself multiple times. Of course, if you need to ask a question in private, do so instead.

Once an RFC is done and has at least one approval, it can be merged. When deciding whose approvals are required for a merge, there is no hard rule - instead think about the teams impacted by this change and try to get at least one representative of each to sign off on this. If you do not know whom to invite, ask anyone in the project for a suitable reviewer.

The prescribed process ends here. How proposed features and changes make it into the product is currently unspecified.

### Free-form

Note that the process is somewhat free-form, this very first RFC already deviates from the template. The latter, like the pirate code, should be seen as more of a guideline than a hard rule.

## Drawbacks

[drawbacks]: #drawbacks

The one danger is that we are just creating a situation where all the existing ways of discussing ideas mentioned in the introduction persist and this just creates yet another one. We are relying on this process being adopted, if that fails, this repo should be dissolved.

Another is potential growth of the process, making it become cumbersome. Any extension to this RFC itself should keep this in mind.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Some of the existing infrastructure could accomodate this process at least partially:

### Wiki/Confluence

The internal confluence or any other Wiki could also handle some of this process, however they are typically lacking the sophisticated change tracking that is inherit in revision control systems like git. While this feature typically comes with a UX hit due to complexity, the fact that all people working on the project are familiar with developer tools makes this a non-issue.

### docs.casperlabs.io

The docs repository is for outside-facing, polished documentation. People reading it are typically not concerned about new ideas that are still under discussion or not implemented yet, and do not want to read about refactorings of node internals.

### Naming

Alternative names were proposed, but rejected in favor of RFC, since it is a well known standard and does not restrict the domain of things discussed. It also does not make us sound self-important.

* `CIP` (Casper Improvement Proposal)
* `CEP` (Casper Enhancement Proposal)
* `CRC` (Casper Request for Comments)

## Prior art

[prior-art]: #prior-art

The process is heavily based on [Rust's RFC process](https://github.com/rust-lang/rfcs) process, which seems to have worked well for the project. Other projects like NEAR [have adopted a copy of it](https://github.com/nearprotocol/NEPs/) as well.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

- Is RFC okay as a name, or do we need to disambiguate it with a custom name?

## Future possibilities

[future-possibilities]: #future-possibilities

With a process in place, future work could include automating some tooling around the process, i.e. creating an automatic [mdbook](https://github.com/rust-lang/mdBook) and publishing it to GitHub pages. These things are not tackled in this RFC to keep it small and get the ball rolling quickly.
