**Black Trans Lives Matter**. Before you get started here, consider setting up a monthly repeating donation to a non-profit such as [LGBTQ Freedom Fund](https://www.lgbtqfund.org/).

# Running Enzyme tests in both React and Preact modes

Etsy is in the process of migrating from React v15.6.2 to Preact v10. We reached that decision after a lot of research and discussion, detailed [here](preact-vs-react.md).

Migrating a codebase as large as Etsy's from one library to another is a significant undertaking. Using `find . -name "*.[jt]sx" | xargs wc -l` in our JS directory shows ~250,000 LOC across 2700 files. There are a very large number of components which use best practices from multiple "eras" of React development at Etsy, and the code which is most out-of-date (compared to our current choices) is often found in the most important areas of Etsy's React codebase. Thankfully, Etsy invests heavily in automated testing, and our testing coverage is generally very reliable. Because of this, the first step in our migration is that our Jest+Enzyme testing suite has to pass when we are using Preact. However, because our migration will not happen overnight, we cannot stop running our test suite in React.

Now that our test suite has finally gotten to green, this post will cover the bits of infrastructure and helpers which were created to make it easy for our developers (and our CI tests) to run Jest in both React mode and Preact mode.

A slow migration between multiple test frameworks involves asking a lot of patience of your users (your coworkers!). Because of this, ensuring that the process be as painless as possible was the number one concern. For that reason, developer ergonomics were a big concern when setting up these changes -- it was more important to ensure that the changes would be easy to use than it was to ensure that the underlying implementation was straightforward. While we worked behind the scenes to get all of the tests passing for both frameworks, we did not want to disrupt the process that other engineers used.

## Running Tests

### Methodology

One of the most important parts of running tests is that they be easy to run. At the same time, when we were finally ready to require that a given test file works for both React and Preact, that too needed to be easy to do.

When running Jest from the command line, our developers would generally issue the following command:
```
yarn jest <maybe-a-path-here>
```

To simplify running tests in Preact mode, we added the following entry to the `scripts` section in our `package.json` file:
```
"scripts": {
  ...
  "jest-preact": "PREACT=1 jest",
  ...
}
```

Note that we haven't yet talked about how that `PREACT=1` is used behind the scenes. Since developer UX was the most important concern, picking how the tests would be _run_ was more important than how running them would work under the hood.

I had originally wanted to use a custom flag passed to Jest to handle this, but it turns out that Jest does not pass custom flags on to its setup files -- trying to do something like `yarn jest --preact-mode` wouldn't work. Environment flags, on the other hand, were quite straightforward to access during test setup and runtime. From a developer UX perspective, this also had another important side-effect: code editors like PHPStorm allow for the setting of global environment variables in test runs, so it was very easy to switch between always running React or always running Preact for tests.

### Implementation

Etsy's test-running environment is complex enough that we use `jest.config.js` rather than specifying Jest setup options in a JSON file.

To start, we needed a single source of truth for whether Jest was running in Preact mode or React mode. That can be seen here:
```js
// <root>/jest-config/helpers/isPreact.js
module.exports = process.env.PREACT === "1";
```

And then in our `jest.config.js`, the actual code which handles most of the "magic":
```js
// <root>/jest.config.js
...
// Jest doesn't allow developers to write their own custom params for the CLI, so we need to specify this via
// ENV variable.
//
// When jest is run with the PREACT variable, all of our react and react-testing functionality will be aliased to use
// Preact
const shouldAliasPreact = require(path.join(JEST_CONFIG_ROOT, "helpers/isPreact"));
const preactAlias = shouldAliasPreact
    ? {
          "^react$": "@preact/legacy-compat",
          "^react-dom/test-utils$": "preact/test-utils",
          "^react-dom$": "@preact/legacy-compat",
          "^enzyme-adapter-react$": "enzyme-adapter-preact-pure",
      }
    : { "^enzyme-adapter-react$": "enzyme-adapter-react-15" };
...
    moduleNameMapper: {
        ...preactAlias,
    },
```

One of our explicit goals is that developers shouldn't have to worry about whether they are running in Preact or React mode. In order to accomplish that, they need to be able to use `import("react")` and have it work no matter the framework. Moreover, we wanted to make the Preact/React logic as invisible to the rest of the environment as possible. The original `alias` section comes from [the Preact docs](https://preactjs.com/guide/v10/getting-started#aliasing-in-webpack). Additionally, one of the necessary bits of [Enzyme setup](https://enzymejs.github.io/enzyme/#installation) is to use the proper "adapter" for your framework. Since we could be using Preact or React, we added an alias for `enzyme-adapter-react` (a library which does not currently exist) and ensured that it would resolve as the proper adapter based on which framework we're using.

Enzyme's [installation instructions](https://enzymejs.github.io/enzyme/docs/installation/) guide you to create a setup file. Ours is included in our `jest.config.js` as such:
```js
    setupFilesAfterEnv: [
        // Configure enzyme for React testing. Must be run _after_ env is set up
        `${JEST_CONFIG_ROOT}/setup-files/enzyme.js`,
    ],
```

And the actual file looks like:
```js
// <root>/setup-files/enzyme.js

// Enable preact/debug for all Jest tests. Shouldn't cause any issues for React tests.
require("preact/debug");

const enzyme = require("enzyme");

// Note: `enzyme-adapter-react` is not a real package.
//
// Instead, in order to allow developers to override this value to go with different versions
// of react, we require in an alias here. In the root jest.config.js file, this alias maps to
// the same version of enzyme-adapter-react that is used for our current version of react.
const Adapter = require("enzyme-adapter-react");

// note that the adapters don't export with the same format
enzyme.configure({ adapter: new (Adapter.default || Adapter)() });
```

So with all of these pieces added to our existing setup, we were now able to run tests using Preact by typing `yarn jest-preact` instead of `yarn jest`.

## Isolating tests to a specific framework

### Methodology

While we aimed for getting every test to pass with both frameworks, we knew from the start that that would not be possible. Whether due to differences in how the Enzyme adapters work or in how the underlying frameworks function, aiming to have every single test block pass for both React and Preact was an infeasible goal. Moreover, trying to accomplish this would be quite frustrating for developers. For this reason, we augmented Jest's `describe` and `it` methods so that a given test (or block of tests) could be set up to only run in React or only in Preact. While we strive to use this as little as possible, there are times when it was unavoidable. In our codebase, they're called `.reactSkip` and `.preactSkip`.

### Implementation
```js
// <root>/jest-config/setup-files/preact-setup.js
const jestGlobals = require("@jest/globals");
const isPreact = require("../helpers/isPreact");

const { describe, it } = jestGlobals;

const dSkip = describe.skip.bind(describe);
const iSkip = it.skip.bind(it);

// Describe blocks with this function will be skipped when running as Preact
describe.preactSkip = isPreact ? dSkip : describe;
// Describe blocks with this function will be skipped when running as React
describe.reactSkip = !isPreact ? dSkip : describe;
// It blocks with this function will be skipped when running as Preact
it.preactSkip = isPreact ? iSkip : it;
// It blocks with this function will be skipped when running as React
it.reactSkip = !isPreact ? iSkip : it;
```

And then in `jest.config.js`:
```js
...
    setupFilesAfterEnv: [
        // Note that this has to run before the enzyme setup.
        `${JEST_CONFIG_ROOT}/setup-files/preact-setup.js`,
        // Configure enzyme for React testing. Must be run _after_ env is set up.
        `${JEST_CONFIG_ROOT}/setup-files/enzyme.js`,
    ],
```

Etsy is currently in the process of adopting TypeScript, and we don't want folks who try to use `.reactSkip` and `.preactSkip` to run into TS errors. To resolve that, we added to our project's `@types` directory a `jest.d.ts` file:
```ts
declare namespace jest {
    interface It {
        /** Tests specified with this will not run in Preact mode */
        preactSkip: It;
        /** Tests specified with this will not run in React mode */
        reactSkip: It;
    }

    interface Describe {
        /** Tests specified with this will not run in Preact mode */
        preactSkip: Describe;
        /** Tests specified with this will not run in React mode */
        reactSkip: Describe;
    }
}
```

This utilize the existing Jest types in the same way that `it.skip` or `describe.skip` do.

Finally, critical infrastructure needs test coverage. We already have an existing folder of Jest "helper" files and tests to make sure that we aren't breaking our Jest infrastructure, so we added to that. Note that this file leans _very_ heavily on differences between the React and Preact API -- specifically whether `createPortal` exists. The goal is that if either of our skip functions (on `describe` or `it`) suddenly stops preventing tests from being run, this test file will have test failures.

```js
// <js-root>/jest/__tests__/setup-preact.test.js
/**
 * If a test in this file is failing, it means that the `preactSkip`/`reactSkip` helpers defined in
 * jest-config/setup-files/preact-setup.js are not running properly
 */

import React from "react";
import * as ReactDOM from "react-dom";

describe("preact and react tests", () => {
    describe.reactSkip("this will fail in react mode", () => {
        it("this will fail", () => {
            expect(ReactDOM.createPortal).toBeDefined();
        });
    });

    describe("this will fail in react mode", () => {
        it.reactSkip("this will fail", () => {
            expect(ReactDOM.createPortal).toBeDefined();
        });
    });

    describe.preactSkip("this will fail in preact mode", () => {
        it("this will fail in preact mode", () => {
            expect(React.createPortal).not.toBeDefined();
        });
    });

    describe("this will fail in preact mode", () => {
        it.preactSkip("this will fail in preact mode", () => {
            expect(React.createPortal).not.toBeDefined();
        });
    });
});
```

Note that this test file "abuses" several "knowns" about our test setup to provide test coverage.
- First, this test would not work the same way in a React v16 environment, because `ReactDOM.createPortal` would be defined. If `reactSkip` somehow stopped preventing test blocks from running, those tests would still pass, and thus our coverage would be lost.
- Second, this test knows that in Preact mode, both `react` and `react-dom` will be aliased to `@preact/compat-legacy`. In React v16, `createPortal` would _never_ be defined on the `react` import, just on `ReactDOM`. Were we working in a React v16 environment, the checks would get updated to assert on whether or not `react` includes `createPortal`, and likely ignore `ReactDOM` entirely. Through this, we _also_ gain coverage of whether our aliases in `jest.config.js` are working correctly.

## Conclusion

With all of this built out we can:
- easily run tests in both React and Preact mode
- in an IDE or from the command line
- exclude individual test cases or entire test blocks from being run in either Preact or React
- feel confident that if our infrastructure starts breaking, it won't fail quietly -- specific tests will start failing to give us an opportunity to debug with purpose

With all of this done, it was time to start actually start updating our tests so that they work for both frameworks. That work, however, will be covered in a followup post.
