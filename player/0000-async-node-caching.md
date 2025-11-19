# Overview
[overview]: #overview

- Name: Async Node View Caching
- Platform (Core, React, JVM, Android, iOS): Core
- Date: <!-- Start Date -->

# Summary
[summary]: #summary

Update the caching strategy around async nodes to isolate the work inside the `AsyncNodePlugin` and fix potential issues with storing cache information on the `Node` type itself.

<!-- One paragraph explanation of the feature -->

# Motivation
[motivation]: #motivationz

There are a couple problems with the current setup that need to be handled before async features see wider adoption:

1. The cache information stored in `Node.Node` for `asyncNodesResolved` can be manipulated during any transforms or `beforeResolve` taps that can cause issues if mishandled. While we can document the importance of maintaining the list as-is, it is easier if the cache is handled outside of the nodes and does not need extra work to maintain at any step.
2. Referencing async features within player core does not make sense when the `AsyncNodePlugin` is a separate, opt-in feature

# Design
[design]: #design

## Remove all async node references from the resolver

To continue with the pattern of `AsyncNodePlugin` being its own plugin, separate from the core, the core resolver should not know anything of how the dependencies are managed. This means removing the `asyncNodesResolved` list from the `Node` type and the `AsyncIDMap` used to track changes in the resolver.

## Track changes in the `AsyncNodePlugin`

In the `AsyncNodePlugin`, keep a map of AST nodes to async node ids. The nodes being tracked should be the original tree's nodes, similar to how the check-path plugin needs to track nodes for keeping its tree accurate. Any time an async node is updated it should mark the cache as invalid. To do that it needs to tap into the `skipResolve` of the resolver and return `false` whenever changes are detected. We need to ensure that this return is done not only for the node resolving the async node, but all of its parents in the tree as well. Tracked changes should be untracked or marked as complete

# Risks
[risks]: #risks

- Keeping track of the node tree can be error-prone. As we've seen with the check path plugin, it is possible to lose track of nodes, or have them change between renders. Tracking the original node usually ensures the object reference is kept, but its parent node might not always be accurate depending on how the current node was generated.
- Managing async node changes within the plugin makes it difficult to ensure the timing for an update is correct. Unlike data updates which are grouped and sent in a single view update, the changes here are tracked externally to that update call. We need to ensure that the update is using all async node changes it can, and only stop tracking changes after they have been used.

# Unknowns
[unknowns]: #unknowns

- Does the `AsyncNodePlugin` need to be kept as a separate plugin? What do we have to gain or lose by introducing a built-in `AsyncController` that the `ViewController` can tap into to manage updates?
  - One consideration here is that an `AsyncController` allows us to build in direct dependencies to async updates within the `ViewController` this could let us batch the updates with other data updates and use a similar tracking strategy to what is used now, but without the need for the `asyncNodesResolved` list per `Node`.

# Alternatives Considered
[alternatives-considered]: #alternatives-considered

N/A

# Unlocks
[unlocks]: #unlocks

N/A