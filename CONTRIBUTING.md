# Contributing

## Process for creating a new proposal

1. Create an
   [issue](https://github.com/direct-state-transfer/dst-go/issues/new?template=improvement.md)
   in the dst-go repository, briefly describing the proposal. For
   smaller changes this may be enough. More involved issues may warrant
   creating a detailed proposal.

2. Add a `proposal` tag to the issue. Since not all issues require a
   design proposal, this tag helps to differentiate between the ones
   that have a corresponding design-doc and the ones that do not.

3. Use [this template](design/000-template.md) to create a design-doc,
   and start a detailed discussion of design related aspects of the
   issue created in previous step.  

4. Submit your proposal by creating a pull request to add the new
   design-doc to this repository.

   Naming convention for proposals: `NNN-Title`, where `NNN` is the pull
   request number and Title is a short name, with words separated by a
   hyphen (`-`).

5. A discussion on the issue among the project members will be done
   after this. Once a final outcome is reached, the proposal will be
   closed with one of the following two outcomes: `accept` or `reject`
   and the PR can be merged.

6. If a proposal is accepted, its implementation in the project will be
   followed up by creating an `implementation task` or linking to an
   existing one.

## Legal stuff

### Intellectual Property

Your contribution must be licensed under the Apache-2.0 license, the
license used by this project. By raising a PR and signing off your
commits as described below, you certify that this is the case for your
contribution.

### Sign the CLA and sign your work

We believe that if successful the DST project should move to a neutral
home in the long run (e.g. to some not-for-profit foundation). To allow
us to retain the necessary open source licensing flexibility please
ensure you have signed the [Contributor License Agreement
(CLA)](https://cla-assistant.io/direct-state-transfer/dst-proposal)
before creating any pull request. All contributors will retain ownership
in their contributions while granting us the necessary legal rights. The
CLA only needs to be signed once and it covers all [Direct State
Transfer](https://github.com/direct-state-transfer) repositories.

This project also tracks patch provenance and licensing using
Signed-off-by tags. With the sign-off in a commit message you certify
that you authored the patch or otherwise have the right to submit it
under an open source license and you acknowledge that the CLA signed for
any DST project also applies to this contribution. The procedure is
simple: just append a line

    Signed-off-by: Random J Developer <random@developer.example.org>

to every commit message using your real name or your pseudonym and a
valid email address.

If you have set your `user.name` and `user.email` git configs you can
automatically sign the commit by running the git-commit command with the
`-s` option.  There may be multiple sign-offs if more than one developer
was involved in authoring the contribution.
