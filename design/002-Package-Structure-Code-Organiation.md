<!-- This is a template for proposing design changes to the dst-go project. -->

# Proposal: Package structure and code organization for dst-go

* Author(s): Manoranjith
* Status: accept
* Related issue: [dst-go#38](https://github.com/direct-state-transfer/dst-go/issues/38)

<!-- Use the above format for issues on github and full links for issues on other platforms. -->

## Summary

<!-- Provide a tl;dr summary -->
Propose a structure for organizing packages in the project to achieve the
following objectives: good readability, loose coupling of dependencies, and
good testability.

## Motivation

Architectural lessons learned from the previous implementation of
[dst-go](https://github.com/direct-state-transfer/dst-go/tree/legacy/master).

Also, since the project is being developed from scratch, having a well defined
way to organize packages and code can provide clarity and common understanding
for all the collaborators.

## Details

<!-- Provide a detailed description of the proposal. -->
1. The common types and interfaces used in the project are defined at a
   root level package `dst`.  This package should not import any other package
   in this project and should contain only definitions.

2. A separate package is used for implementing the functionality that depends
   on an external system (E.g., contacts provider, communication protocol).

3. If the functionality has multiple implementations, use a separate package
   for each of the implementations, with all such packages located in the same
   parent directory.  E.g.: `comm/tcp`, `comm/websocket`.

4. Tests should be added in "_test" package, so that all tests are
   written against the public API (exported functions / types) of the
   package. Test only the expected behavior and not the implementation
   details.

5. If a package needs to implement test helpers for use by other packages, put
   this out in a separate package "packagetest", located in a sub-directory of
   the package. E.g., `node`, `node/nodetest`.

6. Use mocks only for failing tests. Use some real implementation or test
   helpers for happy path tests. Put all the generated mocks in the
   `internal/mocks` package at root level.

7. Wire all the dependencies together in the main package. In this project it is
   `node` package.

Resulting package structure for
[dst-go](https://github.com/direct-state-transfer/dst-go) is shown in the
diagram below.

![Package Structure image not found](002/package_structure.png?raw=true "Package structure")

## Rationale

<!-- Provide a discussion of alternative approaches and trade offs; advantages
and disadvantages of the specified approach.  -->

### Readability

1. By having the common types at root level `dst` package, the types and their
   usage become more meaningful. E.g., `dst.Peer`, `dst.User` vs. `user.User`,
   `peer.Peer` and `var user dst.User` vs `var user user.User`.

2. By organizing packages based on functionality and having a dedicated package
   for different implementations of the same functionality grouped under the
   same parent sub directory, it becomes easy to locate the code corresponding
   to any particular concern.

### Loosely coupled dependencies

1. By defining the expected behavior from a component as an interface in the
   root level package and writing tests against the expected behavior, the
   different implementations of same functionality can be substituted for one
   another.

2. It also enables the different implementations of the same functionality to
   be developed and maintained independent of each other.

### Testing

1. By using real implementations and exported test helpers for happy
   path tests, deep mocking is avoided. Often deep mocking couples the
   tests with the implementation details of the external component used.

2. By using mocks to fail tests, it is easier to write negative test
   cases. Simple one level mocking would be sufficient in this case as
   the mock can be configured to behave in unexpected manner. See
   `client/client_test.go` for an example of this.

3. By writing tests against the external behavior and not the
   implementation, the code can be refactored easily without breaking or
   changing the tests. Tests are added only when there is additional
   expected behavior or change in expected behavior.

4. Since unit tests ensure the components meet the expected behavior, the
   project would require lesser amount of integration tests.

#### Alternate approaches

1. Another approach to define common types is to have no root level package and
   put them in the dedicated package implementing the functionality.
   E.g., CommBackend type would be defined in `comm` package instead of root level
   `dst` package.  
    * By defining the interfaces and dependencies at a root level
      package, there is a clear separation between the definition of
      expected behavior and the implementation of it.  
    * It also prevents compile time dependency between the different
      component packages. See the diagram, all component packages
      depend on the root level package and only the main package
      depends on the component packages. Exception is the dependency
      of client on ethereum on-chain package, because the client is
      coupled to this on-chain implementation and it cannot be substituted.

2. Another approach to organize the different implementations of same
   functionality is to put them each in a separate file in the same package.
   While this approach results in fewer number of packages, the code in a single
   file would implement many details of the functionality and the file can become
   large and less readable.


## Impact

<!-- Choose the level of impact this proposal will have: -->

<!-- Minor (Does not impact any existing features) -->
<!-- Major (Breaks one or more existing features) -->
<!-- New Feature (Introduces a functionality) -->
Architecture (Requires a modification of the architecture)

## Implementation

<!-- Provide a description of the implementation aspects. -->
Coding guidelines need to be added for the project.
