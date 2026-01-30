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

## Improve change tracking in view resolver to do what I want it to do

### 1. Allow resolver update to take Set of changed nodes
This replaces the set of async node ids and will be checked during resolution time to decide which parts of the cache should still be used

### 2. Update `populateFromCache` to check if children should skip cache too
Each step of `populateFromCache` should check both the `skipResolve` and the new `Set` of changed nodes to see if the node should use the cache or not. If any of the nodes should not use the cache then exit the `populateFromCache` function and continue resolving as normal. The results from checking the `Set` and `skipResolve` can be stored in an object kept with the scope of the resolver update so that the hook call doesn't need to be called many times for each `Node`.

### 3. Update `ViewController` to allow changes to be queued from elsewhere
The `View` itself has an `updateAsync` function that is being used, but in order to batch content changes with other view changes, the function should be moved to the `ViewController` with some additional updates. The new function can act as a `markAsChanged` function on the `viewController` and take a `Set<Node>` that represents the updated nodes in the view. The `ViewController` can use its internal `queueUpdate` function to batch these changes with data changes.

After this change is made, the `AsyncNodePlugin` will need to be updated to make use of this new function.

# Risks
[risks]: #risks

# Unknowns
[unknowns]: #unknowns

N/A

# Alternatives Considered
[alternatives-considered]: #alternatives-considered

## Add `AsyncController` into core player

Replace `AsyncNodePlugin` with a built-in `AsyncController` that the `ViewController` can tap into to manage updates, similar to how it interacts with the `DataController`.

### Reason this was dropped
Having updated change tracking on the resolver solves the same issue this was trying to solve, and keeping the `AsyncNodePlugin` as its own plugin allows the features to stay opt-in.

## Track changes in the `AsyncNodePlugin`

In the `AsyncNodePlugin`, keep a map of AST nodes to async node ids. The nodes being tracked should be the original tree's nodes, similar to how the check-path plugin needs to track nodes for keeping its tree accurate. Any time an async node is updated it should mark the cache as invalid. To do that it needs to tap into the `skipResolve` of the resolver and return `false` whenever changes are detected. We need to ensure that this return is done not only for the node resolving the async node, but all of its parents in the tree as well. Tracked changes should be untracked or marked as complete

### Reason this was dropped

This is a lot of work to expect the plugin to track, and is not a good enough pattern to be recommended for other plugins that need to track and make changes to specific pieces of content. 

# Unlocks
[unlocks]: #unlocks

N/A