# Adopt the CEP process

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0001](https://github.com/casperlabs/ceps/pull/0001)

Make specifications, ideas and implementation guidelines more developer friendly and accessible through the introduction of an easy to use CEPs process.

## Motivation

[motivation]: #motivation

To standardize on a single unified process for capturing and specifying proposals for enhancements to our software. 

There is a need for a reasonably lightweight process to discuss and adopt ideas, not only on the tech side, but governance and economics as well. We want this process to serve those behind these ideas and those implementing them first, instead of being caught in project management tooling. If possible, it should also be friendly to outside contributors, setting a low barrier for them to contribute feedback or their own ideas.

Typical usecases are the discussion of refactoring or design decisions of the software, policies for governance, or schemes cooked up by the economics team.

By introducing a little bit of formality, we hope to keep these designs fun and stave off the introduction of more restrictive processes. Creating CEPs should be easy using Markdown and GitHub, tools that everyone is already familiar with.

Largely based on the [Rust RFC process](https://github.com/rust-lang/rfcs), albeit simplified, this document will tell you all you need to know about creating a CEP.

## How to create a CEP

[guide-level-explanation]: #guide-level-explanation

Any CEP (short for "casper enhancement proposal") starts out with an idea. Some ideas are small enough to be exhaustively discussed in a short Slack conversation before making it into a pull request, there is no need to create a CEP for these. However, after some initial vetting, discussion should be moved into the CEP format using the following process:

1. Fork the CEP repo at [casperlabs/ceps](https://github.com/casperlabs/ceps).
2. Create a new branch for your CEP on your private repo, name it accordingly, e.g. `my-new-proposal`.
3. Copy the `0000-template.md` from the root to `text/0000-my-new-proposal.md`.
4. Edit the file, creating the first draft of the CEP.
5. Once the proposal is ready to be discussed, create a pull request to the CEP repo at [casperlabs/ceps](https://github.com/casperlabs/ceps). This PR will have a number, which is the official CEP number.
6. Add one commit immediately that updates the file name and links inside the CEP with the assigned number. Afterwards, add a "Rendered" link pointing to the branch-latest file via GitHub on your CEPs branch for easier reading (e.g. `https://github.com/yourgitusername/CEPs/blob/my-new-proposal/text/1234-my-new-proposal.md`)

### Discussion and merging

The CEP now enters the discussion period. Invite people to review it by requesting reviews and/or messaging them directly. The ensuing discussion may result in proposed changes, which should be addressed through additional commits.

Feedback and comments to the PR should be handled via GitHub to capture the discussion for later reference.

The initial guidance for acceptance is that once a CEP is finished and has at least two approvals from core developers it can be merged. As the project and this process matures, this guidance is expected to be replaced with a more formal requirement.

The prescribed process ends here. How proposed features and changes make it into the product is currently unspecified.

### Free-form

Note that the process is somewhat free-form, this very first CEP already deviates from the template. The latter, like the pirate code, should be seen as more of a guideline than a hard rule.

## Drawbacks

[drawbacks]: #drawbacks

The goal of this process is to unify and replace previous approaches on other content sharing platforms. If the other previous approaches are not abandoned in favor of this one, this attempt at unification will effectively be dead on arrival having merely exacerbated the problem it was meant to solve.  

Another is potential growth of the process, making it become cumbersome. Any extension to this CEP itself should keep this in mind.

While suggested, it is not possible to make the branch contain the CEP number, since it is assigned only by the time the PR is made, for which the branch already needs to exist. This is also why the seperate step updating the docs is required.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Some of the existing infrastructure could accomodate this process at least partially:

### Wiki/Confluence

The internal confluence or any other Wiki could also handle some of this process, however they are typically lacking the sophisticated change tracking that is inherent in revision control systems like git. While this feature typically comes with a UX hit due to complexity, the fact that all people working on the project are familiar with developer tools makes this a non-issue.

### docs.casperlabs.io

The docs repository is for outside-facing, polished documentation. People reading it are typically not concerned about new ideas that are still under discussion or not implemented yet, and do not want to read about refactorings of node internals.

## Prior art

[prior-art]: #prior-art

The process is heavily based on [Rust's RFC process](https://github.com/rust-lang/rfcs) process, which seems to have worked well for the project. Other projects like NEAR [have adopted a copy of it](https://github.com/nearprotocol/NEPs/) as well.

## Future possibilities

[future-possibilities]: #future-possibilities

Future work could include automating some tooling around the process, i.e. creating an automatic [mdbook](https://github.com/rust-lang/mdBook) and publishing it to GitHub pages. 
