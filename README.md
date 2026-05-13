# Casper Enhancement Proposals

CEPs are proposals for improvements to the Casper node, related crates or the surrounding ecosystem. Anyone is free to submit a CEP, but please read the following section on how to create a great proposal:

## How to create a CEP

Any CEP starts out with an idea. Some ideas are small enough to be exhaustively discussed in a short Slack conversation before making it into a pull request, there is no need to create a CEP for these. However, after some initial vetting, discussion should be moved into the CEP format using the following process:

1. Fork the CEP repo at [casper-network/ceps](https://github.com/casper-network/ceps).
2. Create a new branch for your CEP on your private repo, name it accordingly, e.g. `my-new-proposal`.
3. Copy the `0000-template.md` from the root to `text/0000-my-new-proposal.md`.
4. Edit the file, creating the first draft of the CEP.
5. Once the proposal is ready to be discussed, create a pull request to the CEP repo at [casper-network/ceps](https://github.com/casper-network/ceps).
6. You have two choices to number your CEP. Either: (a) the number of the PR you just created, or (b) a meaningful number that indicates alignment with a known concept outside of the Casper ecosystem (e.g., CEP-1155 as the Casper port of ERC-1155). If using a non-PR-based number, you must reference the external concept as prior art in the CEP.
7. Add one commit immediately that updates the file name and links inside the CEP with the assigned number, when basing it on the PR number. Afterwards, add a "Rendered" link pointing to the branch-latest file via GitHub on your CEPs branch for easier reading (e.g. `https://github.com/yourgitusername/CEPs/blob/my-new-proposal/text/1234-my-new-proposal.md`)

## History

CEPs were introduced using [CEP0001](text/0001-adopt-cep-process.md).
