**Black Trans Lives Matter**. Before you get started here, consider setting up a monthly repeating donation to a non-profit such as [LGBTQ Freedom Fund](https://www.lgbtqfund.org/).

# Updating React at Etsy

This post is based on technical documentation I put together as part of work to research and summarize the pros/cons of upgrading Etsy's default React version from v15.6.2 to either Preact v10.X or React v16.13 (the latest versions of each at time of writing). While there are many variations on "should I use Preact or React", this is ours. There are many reasons why a team would choose one library over another, so I won't pretend that this covers the full set of them -- just the ones which felt extremely relevant to making a decision which was right for Etsy. One important piece of context is that Etsy currently has two major product stacks. For buyer-facing pages, we use PHP server-based rendering with jQuery/vanilla JS on the client to stitch things together. For our seller-facing pages and many of our internal tools, we use React-rendered SPAs with minimal server-based HTML rendering, receiving data from the same PHP server-side stack.

Because the post ends up being quite long and fairly in-depth, I had mercy on my fellow developers and presented the summary first. Not everyone needs or wants to go deep on figuring these things out, and one of the benefits of working at a company that is Etsy's size is that we can afford to have a smaller number of people who specialize in different parts of our stack. I, specifically, spend a lot of time working on the low-level bits that make life easier for the rest of Etsy's developers to do product work and ship features/fixes/good code.

This doc was shared internally with a survey in order to ensure that I was not dictating the decision for a huge number of engineers. The results were, happily, that Etsy will be upgrading from React v15.6.2 to Preact v10.X. Now, enough with the extra commentary -- here's the doc.

## Summary of Findings

In my opinion, Etsy should migrate from React v15.6.2 to Preact v10.X (latest).

Preact is an alternate implementation of the React API. Migrating from React to Preact is a fairly straightforward process -- this would not entail a huge rewrite or significant work from any other teams.

This is because:
- The [FES](#fes) team has already moved forward with a decision to investigate Preact-based development in the buyer-side of Etsy’s codebase with SSR. Choosing React over Preact for seller tools would increase the fragmentation of our development environment and make our lives harder.
- The Preact v10.4.2 bundle is **6KB gzipped**, vs **38.5KB gzipped** for React v16.13.1.
- The migration to Preact v10.4.2 would be significantly easier and require far fewer steps to complete, due to Preact’s emphasis on compatibility with both React v15 and React v16.
- There do not seem to be any major obstacles from a developer tooling perspective to adopting Preact.

That said, the goal of this document is to give everyone the information they need to form their own opinion, so that we can all make a decision together.

## Introduction

[Orange docs](#orange-docs) at Etsy seem like project descriptions, but this document instead represents compiled research, presented in such a way that we as a developer community at Etsy can make a decision on which way we would like to go when upgrading our version of React. Upgrading React from v15.6.2 brings significant performance and bundle size improvements while also providing a lot of new features. Migrating from React v15 to React v16 is a major undertaking. There are a number of backwards incompatible changes which must occur within the codebase in order to enable these improvements. 

While it would be really hard to describe here in detail all of the new features, the code architecture improvements enabled by the React v16.8 [Hooks](https://reactjs.org/docs/hooks-intro.html) functionality are so significant that it will eventually become harder and harder to recruit developers interested in working in a pre-Hooks codebase. In order to keep up with the rest of the industry and provide our current developers with the best tooling available, I believe it should be a high priority to enable use of Hooks and other modern React functionality. This definitely isn’t anywhere close to the only new feature worth upgrading for, but it does represent an extremely large shift in how developers will be able to write React code.

At the same time, there is a growing recognition within the web development community that the large JS bundle size of React’s libraries negatively impacts users, for all that it can make code significantly easier to understand and maintain over time. The FES team has already adopted [Preact](https://preactjs.com/), a tiny drop-in alternative to React which is API compatible and would not involve a major rewrite to adopt in our seller tools. They have specific documentation intended to help with the (relatively few) differences between Preact and React. It can be found [here](https://preactjs.com/guide/v10/differences-to-react/).

## Bundle size improvements

Note that these numbers exclude the size of packages which would be included by all three libraries -- things like `prop-types` or `create-react-class`

Sizes are all specified as minified / minified+gzipped and come from Bundlephobia

React v15.6.2:
- react: 21.4KB / 7.1KB
- react-dom: 121.4KB / 36.6KB
- total: 142.8KB / 43.7KB

React v16.13.1:
- react: 6.3KB / 2.6KB
- react-dom: 114.6KB / 35.9KB
- total: 120.9KB / 38.5KB

Preact v10.4.5
- preact: 10.1KB / 4KB
- preact/compat: unknown / 2KB (the minified size of preact/compat is not available anywhere I can find)
- total: ? / 6KB

To summarize:
- Updating from v15.6.2 to v16.13.1 would save us 5.2KB in gzipped size
- Updating from v15.6.2 to Preact v10.4.5 would save us 37.KB in gzipped size
- Using Preact v10.4.5 over React v16.13.1 would save us 32.5KB in gzipped size

## Preact

Because React is the “standard” tool and Preact is responsible for maintaining its own compatibility, a lot of the discussion around Preact involves compatibility with existing libraries and code. In general a lot of discussion of “Preact compatibility” is somewhat complicated. There are two types of Preact development -- one in which the React compatibility layer is not added, and one in which it is.

Preact v10 merged the compatibility layer into the main Preact stack, and a lot of React-based libraries appear to have gained significant improvements in compatibility because of this.

### Compatibility with Existing Code

Because Preact emphasizes compatibility with both the v15 and v16 React APIs, it does not appear that we would need to do significant work to perform this upgrade. As with React, we need to complete the migrations from `React.createClass` to the `create-react-class` package and `React.PropTypes` to the `prop-types` package. Once that is complete, however, it appears that we would not need to perform significant changes to the codebase to migrate to Preact. We would want to make these changes eventually, but being able to make them slowly and in a controlled fashion would make the update significantly safer.

The FES team’s new project architecture is already based on Preact, which introduces a significant compatibility win by choosing Preact -- ensuring that we use only one Preact / React library for all of Etsy would greatly reduce developer difficulty over time. Among other things, having to support / test in React and Preact for tools like the [Web Toolkit](#web-toolkit) would add a lot of overhead for that team and others working on shared tooling and architecture. This would be avoided if Preact were the standard across all of Etsy.

### Compatibility with Existing Libraries
#### Redux v3.4.0 (**possibly no changes required**)

No direct dependence on React or Preact. If we don't have to upgrade our other libraries, we won't have to upgrade Redux (but we'll need to if we upgrade React-Redux)

#### React-Redux v4.4.5 (**possibly compatible as-is**)

As of Preact v10, use of preact-redux has been deprecated in favor of full support for React-Redux. Our version of React-Redux is quite old, so it’s unclear whether we need to upgrade this library in order to enable that full support.

#### React Router v2.3.0 (**possibly compatible as-is**)

Preact documentation suggests full support by react-router of Preact with the compatibility layer included. That said, it is unclear whether Preact as-is would be compatible with our current version of React Router, or whether we would need to complete a version update to handle this. We can upgrade to at least React-Router v5.2.0 without upgrading React, so this could be done as an independent train of work.

#### React Router Redux v4.0.5 (**possibly compatible as-is**)

Use of this library has been deprecated and it will not work with modern versions of React Router. If we have to upgrade React-Router, we will have to remove this library. Unclear whether it would be compatible with Preact as-is, but it seems likely that it would be compatible.

It’s possible that we can replace this library with `connected-react-router`, which is one recommended alternative. Of note: `connected-react-router` v5.0.1 has a bundle size of 27.4KB / 4.6KB gzipped, whereas `react-router-redux` was 4.1KB / 1.4KB. It also depends on react-router at least v4.3 and react-router v4.3. `connected-react-router` v6.8.0 has a bundle size of 9.5KB / 3.2KB, which is a much better number. Unfortunately, `connected-react-router` v6+ requires React v16.4.0 and React Redux v6 or v7. To summarize, adopting this library would require a significant (though somewhat temporary) additional bundle size hit and add additional complexity to the upgrade process. Additionally, `connected-react-router` appears to have a hard dependency on `immutable`, which would in the long-term prevent us from migrating away from `immutable`.

#### Final Note on Compatibility

As pointed out by the FES team, Preact brings with it an extra overhead -- every library we use will need some extra testing to confirm compatibility. That said, Preact’s emphasis on compatibility should hopefully reduce problems here. 

### Compatibility with Existing Tooling

#### Yarn/NPM versioning

Because we would be using Preact instead of React, we would potentially cause warnings to be printed out by Yarn during installs if we completely removed React from our package.json (unknown). This should not be a true blocker, but semver compatibility is a very useful tool for maintainers and it would be good to have ours working.

We would potentially want to have both React and Preact listed in our package.json file to quiet these issues (unknown)

#### Jest / Enzyme

An [adapter](https://preactjs.com/guide/v10/unit-testing-with-enzyme) is available for Enzyme.

There are some slight differences, but due to our emphasis on `mount`-based testing we should avoid most of the complexities. There may be some issues with `.simulate` tests in Enzyme -- because Preact dispatches real DOM events, the testing surface may expose issues around bubbling which were not present in React tests (but would in theory be present in live code).

#### Jest / [Unitcards](#unitcards)

Unclear, further testing required. In theory, we would update the jest-unitcard runner to implement the preact alias and confirm that everything passes.

#### DevTools

Preact has its own browser extension (which is not the React one), but provides similar debugging functionality and performance monitoring. It can be found [here](https://preactjs.com/guide/v10/debugging/).


### Compatibility with Future Plans

Would adopting Preact impact any future development plans we've discussed for next 1/2/5 years?

#### Typescript support

No specific problems found, appears to have similar compatibility to React's support of Typescript.

#### Suspense?

Suspense is still experimental for React, and seemingly more experimental for Preact.

True SSR Suspense support would probably involve pretty significant rewrites to a massive amount of our codebase, due to the changes in the way that components work. While it might be possible to adopt Suspense on new development and in the toolkit, it's likely not feasible to adopt it in existing code without a large-scale effort.

#### GraphQL

`@apollo/client` appears to support Preact, but nothing specific is listed. `urql` explicitly supports Preact. As true clientside GraphQL support is not available, it's not yet possible to say definitively that we would have full support. That said, having identified two libraries which would provide what we need (depending on what route we eventually choose in terms of using GraphQL on the client) seems sufficient to say that this is not a blocker.

#### Migrating to React in the future

One concern with choosing a library is whether we can rely on it to remain supported. React is actively supported by Facebook, and that seems unlikely to change in the next few years. Preact is actively supported by donations and maintained by a passionate group of developers, but if we choose Preact we also need to be prepared to migrate from Preact to React if the library stops receiving new development or if we find a showstopping feature which requires React to implement.

- Preact is API compatible with React, which means that we are not making any changes by using Preact which would be incompatible with migrating to React at a later date.
- However, because of Preact’s emphasis on compatibility with React v15 as well as React v16, there are a number of code updates we won’t _have_ to make if we move to Preact. We should do them anyways, but the process will be slow and require refactoring a fair amount of code in somewhat tricky places.
- If we complete these refactors while on Preact and move our codebase to an entirely React v16-compatible use of the APIs, then moving to React v16 from Preact will be very straightforward -- one or two automated code changes that are done via codemod, but otherwise minimal work.
- One original plan for actually completing the React v16 upgrade in a way that wasn’t so complicated was to migrate the entire codebase to Preact, update all of the APIs in place, and then move to React. That ended up being a lot of unnecessary churn with low-level libraries, so we moved away from that idea.

### Migration Plan

This work is currently blocked by the removal of `React.PropTypes` and `React.createClass`, both of which are blocked by the [Web Platform](#web-platform) team's [ESM syntax upgrade](#esm).

Note that most of this work is complex and even a single line / upgrade could involve significant amounts of time. Among other things, the `react-router` breaking changes will require some invasive changes to the seller tools [subapp architecture](#subapps) to rework how things are loaded / routed. Removing `react-router-redux` will also involve a lot of work, though each piece should hopefully be relatively independent.

Assuming that all of our library compatibilities are as stated above and Preact compatibility works as expected, our path to migration would be:
1) [Upgrade to Preact](https://preactjs.com/guide/v10/getting-started/#aliasing-react-to-preact) v10.4.5 behind a switch, so that we can switch between Preact and React powered rendering in order to validate
2) Start applying codemods to migrate to "modern" React lifecycle methods (not a hard requirement, due to full support of both versions' API in Preact, but a good long-term goal)
3) Entirely remove `react-router-redux` (can be done in parallel with the Preact migration)
4) Upgrade `react-redux` to 7.2.0 (likely involves a `redux` upgrade as well)
5) Upgrade `react-router` to 5.2.0 (or possibly v6.x, depending on whether it is sufficiently stable at that point)

## React

Migrating from v15.6.2 to v16.13.1 would require a significant time commitment, due to the large number of breaking changes which have occurred in intervening versions.

### Compatibility with Existing Code

While this is the safest choice in terms of ensuring long-term compatibility, upgrading to React v16 comes with a significant cost. A number of lifecycle methods have been deprecated and renamed, which will require code-mods to be run in order to rename the now-deprecated methods. While we would still eventually want to run these codemods if we choose Preact, in the case of Preact we could migrate fully before making the changes. The React upgrade would require the codemods as part of the upgrade, which makes it harder to undo or work on in pieces.

Because of heavy usage of the now-deprecated [`theseus/Component`](#theseus) helper, the seller tools have relatively little usage of lifecycle methods which are deprecated in React v16. The [Web Toolkit](#web-toolkit), on the other hand, makes use of a large number of these deprecated lifecycle methods, and will require refactoring and regression testing in order to safely be migrated.

Because the FES team's experimental architecture is explicitly Preact based, and because there is a desire to share the Web Toolkit, choosing to migrate to React would impose additional complexity on working in any code which is shared between Preact and React parts of the codebase. While this would initially just be the Web Toolkit, it would also explicitly block our ability to investigate a new version of the seller tools subapp architecture which utilized the Preact SSR service.

### Compatibilites with Existing Libraries
**Brace yourself**.

#### Redux (**upgrade required**)

As we will have to upgrade React-Redux, we will have to upgrade Redux

#### React-Redux (**upgrade required**)
React-Redux v4.4.5 (our current) does not support React v16. React-Redux v7.2.0 (the latest) requires at least React v16.8.3. In order to upgrade React to the latest version, we will need to:
  - Upgrade React-Redux to v5.1.2
  - Upgrade React from v15.6.2 to at least v16.8.3
  - Upgrade React-Redux to v7.2.0

#### React Router (**upgrade required**)

React-Router v2.3.0 (our current) does not support React v16. React-Router only provides upgrade paths to v5 (stable release) or v6 (currently in beta). React-Router v5.2 supports React v15+, so we could perform this upgrade before upgrading to React v16.

#### React Router Redux (**library no longer supported, alternative solution required**)

Use of this library has been deprecated and it will not work with modern versions of React Router. If we have to upgrade React-Router, we will have to remove this library.

It’s possible that we can replace this library with `connected-react-router`, which is one recommended alternative. Of note: `connected-react-router` v5.0.1 has a bundle size of 27.4KB / 4.6KB gzipped, whereas `react-router-redux` was 4.1KB / 1.4KB. It also depends on react-router at least v4.3 and react-router v4.3. `connected-react-router` v6.8.0 has a bundle size of 9.5KB / 3.2KB, which is a much better number. Unfortunately, `connected-react-router` v6+ requires React v16.4.0 and React Redux v6 or v7. To summarize, adopting this library would require a significant (though somewhat temporary) additional bundle size hit and add additional complexity to the upgrade process. Additionally, `connected-react-router` appears to have a hard dependency on `immutable`, which would in the long-term prevent us from migrating away from `immutable`.
    
### Compatibility with Existing Tooling

#### Jest / Enzyme

A React v16 adapter for Enzyme is provided, and no major compatibility / breaking changes appear to result as part of that upgrade.

#### Jest / [Unitcards](#unitcards)

The unitcard framework itself does not appear to rely on functionality which would break in the upgrade, so probably minimal impact here.

#### DevTools

Getting to React v16 would unlock significant new abilities for performance testing in the React dev tools extension.


### Compatibility with Future Plans

Would adopting React impact any future development plans we've discussed for next 1/2/5 years?

#### Typescript support

React team has invested in enabling Typescript support.

#### Suspense?

Our [subapp](#subapp) architecture and lazyloading tools for components already solve many of the problems which Suspense appears to solve, though being able to move to framework-supported tooling rather than in-house tooling would be an advantage. That said, the size of our codebase would provide significant challenges to migrating entirely to Suspense, and there's no guarantee we would see many gains. The `fetch as you render` style encouraged by Suspense has proven repeatedly to require complex state management code to avoid waterfall data fetching patterns and other performance problems, and I've found that reviewing and maintaining these sorts of components can prove to be extremely challenging and time-intensive. 

#### GraphQL

As we are not interested in adopting Relay, the choices remain the same as with Preact: probably either `@apollo/client` or `urql`, depending on the results of experimenting with the resulting architecture to figure out what best fits.

### Migration Plan 

This work is currently blocked by the removal of `React.PropTypes` and `React.createClass`, both of which are blocked by the [Web Platform](#web-platform) team's [ESM syntax upgrade](#esm).

Note that most of this work is complex and even a single line / upgrade could involve significant amounts of time. Among other things, the `react-router` breaking changes will require some invasive changes to the seller tools [subapp architecture](#subapps) to rework how things are loaded / routed. Removing `react-router-redux` will also involve a lot of work, though each piece should hopefully be relatively independent.

- Entirely remove react-router-redux
  - Otherwise we have to, as part of a branch, upgrade both react-router and react-router-redux at the same time. react-router’s upgrade involves significant breaking changes, so this won’t be easy.
- Upgrade react-router to v5.2.0
  - Still a large breaking change involving multiple branches, but at least it can be separate from react-router-redux
- Upgrade react-redux to v5.1.2
- Upgrade React to v16.X (look for a stable React version)
  - Many breaking changes, most of which can be resolved via codemod
  - Note that this will be difficult to roll out slowly and difficult to revert: because so many of the changes will be “breaking”, we won’t be able to as easily switch this on and off -- we’ll need to merge the branch that takes us to v16 with a push hold and do validation on the changes.
- Upgrade React to v16.8.3
  - Codemods + other changes to update lifecycle methods
  - We may want to explore doing v15.6.2 -> v16.8.3 directly without an intermediate step, depending on the scale of the changes from running all of the codemods
- Upgrade react-redux to v7.2.0
- Upgrade React to v16.13.1


# Appendix

In order to avoid mutilating the doc too much by either removing specific teams or trying to explain context inline, some explanation here.

#### FES

Etsy's Front-End Systems team, who build and maintain the systems which our frontend engineers use to develop new products.

#### Buyer Side

Buyer-facing pages at Etsy are currently server-rendered in PHP, with jQuery and other clientside code to stitch them together.

#### Orange Docs

Internal technical documentation with a rigid formula, shared internally in order to seek input on specific architecture or implementation details. 

#### Web Toolkit

Etsy's internal component-based design system. It is currently maintained by our Design Systems team, who are wonderfully patient people, with implementations available both as React components for our seller tools and and vanilla Javascript for our buyer-facing pages.

#### Unitcards

An internally-maintained (deprecated) React component testing framework which predates our (extremely recent) adoption of Jest and Enzyme. We have ~500 unitcard test files performing ~20k assertions all said and done, which make up a significant amount of our React codebase's coverage. Anything which caused us to need to migrate large numbers of tests from the Unitcard framework to Enzyme would be dramatically change the scope of an upgrade project.

#### Web Platform

My team! Where FES is responsible for the tools which power Etsy's frontend rendering and functionality, Web Platfom is responsible for the tooling our developer experience -- things like `yarn`, `eslint`, Webpack, `jest`, polyfills, Javascript feature support, etc. There's enough overlap between the responsibilities of the two teams that we spend plenty of time collaborating with FES, which is great -- they're amazing folks.

#### ESM

Since January, Etsy has adopted Webpack, launched ES6+ support (we were on ES5 syntax with JSX compatibility hacked in to our old build tools as late as March), migrated from Jasmine and Unitcards to Jest and Enzyme, and many other dev tooling upgrades. One of the last major sweeping updates to the codebase left to complete is moving from AMD syntax to ESM syntax. While we're able to use codemods to handle much of this, our codebase and those codemodes have enough incompatiblities that getting to a point where we can experimentally prove that migrating to ESM does not negatively impact users has been quite a challenge. We're almost ready to start this experiment!

#### Subapps

Etsy's seller tools are divided into a central "shared" React app which boots up our sidebar/navigation and **many** mostly-independent "subapps". We use `react-router` in the main app to handle lazyloading each subapp's components, reducers, etc. One of the major goals of this architecture was *independence* -- developers working on the messaging subapp, for example, needed to be able to work on their own without worrying about impacting developers working on analytics tooling or order management. There are some choices made here which might be sub-optimal for a site with a relatively small number of lazyloaded sections, or where each lazy-loaded section was relatively tiny, but our architecture is aimed at scaling out to N subapps (there are currently 32) with significant differences in complexity.

#### theseus/Component

`theseus` is an Etsy-internal framework used as part of a migration to React from our previous stack. Still heavily used as part of our underlying set of helper functions / classes, but being deprecated in favor of external libraries and tooling which is not as dependent on the existing state of our tools.

The `Component` wrapper allowed us to write components which could work in both Backbone and React during the migration, allowing us to swap out parts while the whole remained the same (hence, [the name](https://en.wikipedia.org/wiki/Ship_of_Theseus)). `Component` only allowed usage of a small set of React lifecycle methods and prevented the use of React component local state, among other things.
