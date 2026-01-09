# Overview
[overview]: #overview

- Name: AsyncNodePlugin API Updates <!-- Feature Name -->
- Platform (Core, React, JVM, Android, iOS): Core <!-- Platform(s) affected-->
- Date: <!-- Start Date -->

# Summary
[summary]: #summary

The goal of this RFC is to remove the `AsyncNodePluginPlugin`, update the `onAsyncNode` hook to take a handler object rather than a promise of future content, and to change the node `update` function to provide options that allow for better optimizations around lists of content.

<!-- One paragraph explanation of the feature -->

# Motivation
[motivation]: #motivationz

There are 4 main motivations with these changes:
1. Simplify setup and make any plugin needed for streaming capabilities have a clear purpose.
2. Make interacting with async nodes intuitive, removing confusing promise management as a step for managing control over individual nodes.
3. Introduce an API for dealing with async content that we can more easily make changes to. Minimize the need for breaking changes in the future and simplify the deprecation path.
4. Make managing lists of async content not require inserting many async nodes by providing better optimizations around parsing and resolving changing lists. 

<!-- What, is the motivating factor behind this RFC. -->

# Design

## Removing the AsyncNodePluginPlguin

`AsyncNodePluginPlugin` currently provides most of the features a user would expect from the main `AsyncNodePlugin`. It ends up calling the hooks on `AsyncNodePlugin`, which keeps the features of both tightly coupled to each other. The name has also been a confusing point for users as they expect the `AsyncNodePluginPlugin` to provide some shortcuts to tapping into the `AsyncNodePlugin` to help with managing async nodes.

To simplify this and reduce setup steps, the `AsyncNodePluginPlugin` class can be removed and its features can be moved to `AsyncNodePlugin` so that there are no references across plugins.

`AsyncNodePlugin` can have its list of plugins removed to reduce options in the constructor.

## Changes to the AsyncNodePlugin Hooks

The core design here would be to remove the `onAsyncNodeError` hook, and update `onAsyncNode` to something like:
```ts

/** function type for updating async content. This type matches the current update of `onAsyncNode`. */
export type ContentUpdateFunction = (content: any) => void;

export interface AsyncNodeHandler {
  /** Start function for the handler. Used once on setup to pass the handler the update function and notify it that the node is ready for updates so it can start any async process. */
  start: (node: Node.Async, updateFunction: ContentUpdateFunction) => void;
  /** Function called during the first pass of the resolver, when an async node is initially identified. Setup any placeholder content here. */
  getInitialState?: (node: Node.Async) => any;
  /** Handler for errors during an update triggered by an async node updating */
  onError?: (node: Node.Async, error: Error) => void
}

export type AsyncNodePluginHooks = {
  /** hook to manage handlers for individual async nodes */
  onAsyncNode: SyncBailHook<[Node.Async], AsyncNodeHandler>;
}
```

In this case, the `onAsyncNode` hook has been changed to be a `Sync` hook rather than an `Async` hook. This allows for handlers for each async node to be determined synchronously and gives us the ability to allow for an initial state of the async content without needing to trigger an additional update of the view. This also reduces the burden of promise management on users. Currently in order to ensure only your own plugin is managing a specific node, plugins must manage promises that aren't meant to resolve and call the update function as needed after that. This also allows us to reuse the same `onAsyncNode` hook as we make changes to what the `AsyncNodeHandler` type has down the line.

<!-- A detailed end-to-end design of the proposed system/changes -->

# Risks
[risks]: #risks

- The sync hook approach loses the ability for multiple handlers to exist for a single async node. An alternative here to allow that would be to use a `SyncHook` that provides a `registerHandler` function, but allowing for multiple handlers might make it confusing when and how a specific node is getting updated.
- Managing list operations on the `AsyncNodeUpdater` is always going to be more limited than managing an actual list and will likely require additional support with new options as new use cases come up.


<!-- What are potential issues that could come from implementing this decision -->

# Unknowns
[unknowns]: #unknowns

### How do we avoid breaking changes?

The changes in this RFC, if made directly as suggested, will cause breaking changes in the async node package. To cover for this there are a couple approaches:

#### 1. New Plugin

Instead of changing anything in the existing plugins, mark those as deprecated and make a new plugin within the same package. This is likely to cause some code duplication for now, but does the best job of isolating the change.

#### 2. Change within the plugins

If we want the existing plugins to work with any new changes without having users migrate immediately, this will be more convenient. To support this there are a few things that will need to be done:

- Instead of deleting `AsyncNodePluginPlugin`, mark it as deprecated. Still move all the code to `AsyncNodePlugin` so this essentially ends up being an empty class.
- Do not remove the existing hooks. Add the new API as a new `asyncHandler` hook in addition to the existing hooks. Mark the old hooks as deprecated.
- Mark the existing constructor of `AsyncNodePlugin` as deprecated in favour of something without the `plugins` array argument.

To deal with the overlap between the new `asyncHandler` hook and the `onAsyncNode` and `onAsyncNodeError` hooks, any time one would be called, we should prefer using the result from `asyncHandler` first.

For example, when an async node is first identified and we need a handler, call `asyncHandler` first, and only call `onAsyncNode` if that does not return a result. If an error occurs, check the handler for an `onError` function before calling `onAsyncNodeError` to handle it.

<!-- What, if any, are the open questions that need to be worked through as part of the RFC Process -->

# Alternatives Considered
[alternatives-considered]: #alternatives-considered

N/A

<!-- What, if any, alternate designs/approaches were considered -->

# Unlocks
[unlocks]: #unlocks

N/A
<!-- What, if any, does this RFC unlock in terms of new features/capabilities that might be out of scope for this decision specifically -->

# Out of Scope

## Changing the async node `update` function

There is a lot to consider with this topic and we need more information on how users are using async capabilities before we push for a new implementation on this update function.