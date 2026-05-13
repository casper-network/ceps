# Adopt the CEP process

## Summary

[summary]: #summary

CEP PR: [casperlabs/ceps#0001](https://github.com/casperlabs/ceps/pull/0001)

Make specifications, ideas and implementation guidelines more developer friendly and accessible through the introduction of an easy to use Casper Enhancement Proposal (CEP) process.

## Motivation

[motivation]: #motivation

To standardize on a single unified process for capturing and specifying proposals for enhancements to the Casper software and protocol.

There is a need for a reasonably lightweight process to discuss and adopt ideas, not only to evolve the technology, but also the Casper Network governance and economics. This process serves those behind these ideas and those implementing them first, rather than bogging down innovation cycles with project management tooling. This process should also be friendly to outside contributors, setting a low barrier for external contributors to submit feedback or propose their own ideas.

Typical use cases for the CEP process are the discussion of refactoring or design decisions of the Casper software, policies for governance of the Casper Network, or economic proposals affecting the CSPR token on the Casper Network.

By introducing a consistent format and process, we hope to keep the introduction of new ideas both straightforward as well as enjoyable. Creating CEPs should be created with universally familiar tools: Markdown on GitHub.

Largely based on the [Rust RFC process](https://github.com/rust-lang/rfcs), albeit simplified, this document will tell you all you need to know about creating a CEP.

## How to create a CEP

[guide-level-explanation]: #guide-level-explanation

Any CEP starts out with an idea. Some ideas are small enough to be exhaustively discussed in a short Slack conversation before making it into a pull request, there is no need to create a CEP for these. However, more substantial ideas, innovations and changes are either substantial enough or warrant public pre-disclosure and participation, that discussion should be moved into the CEP format using the following process:

1. Fork the CEP repo at [casper-network/ceps](https://github.com/casper-network/ceps).
2. Create a new branch for your CEP on your private repo, name it accordingly, e.g. `my-new-proposal`.
3. Copy the `0000-template.md` from the root to `text/0000-my-new-proposal.md`.
4. Edit the file, creating the first draft of the CEP.
5. Once the proposal is ready to be discussed, create a pull request to the CEP repo at [casper-network/ceps](https://github.com/casper-network/ceps). 
6. You have two choices to number your CEP. Either: (a) The number of the PR you just made against the formal GitHub repository, or (b) a meaningful number that indicates alignment with a known concept outside of the Casper ecosystem; for example, you may call the Casper port of ERC-1155: CEP-1155. If naming a CEP after a known external concept, you have to reference the external concept as prior art in the CEP. When using a non-PR-based CEP number, the proposed number must be declared in the pull request title and description. The number must be explicitly approved as part of the standard two-core-developer approval process before merge. If the requested number is not approved, the CEP defaults to the pull request number.
7. Add one commit immediately that updates the file name and links inside the CEP with the assigned number, when basing it on the PR number. Afterwards, add a "Rendered" link pointing to the branch-latest file via GitHub on your CEPs branch for easier reading (e.g. `https://github.com/yourgitusername/CEPs/blob/my-new-proposal/text/1234-my-new-proposal.md`)

## How to amend a CEP

CEPs may require amendment from time to time. When introducing materially new functionality, it is generally preferred to create a new CEP. For example, an extension to an existing token standard in most cases warrants its own CEP. However, when changes are either relatively minor or mostly corrective in nature, a previously adopted CEP may go through an amendment process. Similarly, for a CEP that carries a name that is tied to a known external concept, for example a hypothetical CEP-1155 as the Casper-equivalent of ERC-1155, it is appropriate to apply changes to the existing CEP. Amendment adheres to the following process:

1. Fork the CEP repo at [casper-network/ceps](https://github.com/casper-network/ceps).
2. Create a new branch for the CEP you're amending on your private repo, name it accordingly, e.g. `cep-1155-amendment`.
3. Edit the file and make your amendments
4. Once the proposal is ready to be discussed, create a pull request to the CEP repo at [casper-network/ceps](https://github.com/casper-network/ceps).


### Discussion and merging

The CEP now enters the discussion period. Invite people to review it by requesting reviews and/or messaging them directly. The ensuing discussion may result in proposed changes, which should be addressed through additional commits.

Feedback and comments to the PR should be handled via GitHub to capture the discussion for later reference.

The standing guidance for acceptance is that once a CEP is finished and has at least two approvals from core developers it can be merged. 

### Deviations

Deviations from the template should be avoided when possible, but ultimately this is a set of guidelines and a convention; minor deviations are acceptable where called for.

## Drawbacks

[drawbacks]: #drawbacks

The goal of this process is to unify and replace previous approaches on other content sharing platforms. If the other previous approaches are not abandoned in favor of this one, this attempt at unification will effectively be dead on arrival having merely exacerbated the problem it was meant to solve.

Another is potential growth of the process, making it become cumbersome. Any extension to this CEP itself should keep this in mind.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Some of the existing infrastructure could accomodate this process at least partially:

### Wiki/Confluence

The internal confluence or any other Wiki could also handle some of this process, however they are typically lacking the sophisticated change tracking that is inherent in revision control systems like git. While this feature typically comes with a UX hit due to complexity, the fact that all people working on the project are familiar with developer tools makes this a non-issue. Moreover, private wikis do not allow for public participation or external submissions.

### docs.casper.network

The docs repository is for outside-facing, polished documentation. People reading it are typically not concerned about new ideas that are still under discussion or not implemented yet, and do not want to read about refactorings of node internals.

## Prior art

[prior-art]: #prior-art

The process is heavily based on [Rust's RFC process](https://github.com/rust-lang/rfcs) process, which seems to have worked well for the project. Other projects like NEAR [have adopted a copy of it](https://github.com/nearprotocol/NEPs/) as well.

## Future possibilities

[future-possibilities]: #future-possibilities

Future work could include automating some tooling around the process, i.e. creating an automatic [mdbook](https://github.com/rust-lang/mdBook) and publishing it to GitHub pages.
