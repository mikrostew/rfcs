- Feature Name: update_version_handling
- Start Date: 2019-04-23
- RFC PR: [#32](https://github.com/volta-cli/rfcs/pull/32)
- Volta Issue: [#383](https://github.com/volta-cli/volta/pull/383)

# Summary
[summary]: #summary

Change all version-specifying commands to use the `<tool>[@<version>]` syntax from `npm install`, `yarn add`, and `npx <package command>`.

# Motivation
[motivation]: #motivation

Users may regularly want to add multiple tools to their toolchain at the same time. They are also accustomed to being able to install multiple dependencies simultaneously with the `npm install` or `yarn add` commands, both in a package and globally.

-   **Local installs:**

    ```sh
    npm install ember-cli@3.9.1 typescript@3
    ```
    
    ```sh
    yarn global add ember-cli@3.9.1 typescript@3
    ```

-   **Global installs:**

    ```sh
    npm install --global ember-cli@3.9.1 typescript@3
    ```
    
    ```sh
    yarn global add ember-cli@3.9.1 typescript@3
    ```

Volta currently does not support this, requiring individual invocations for each tool:

-   **User toolchain:**

    ```sh
    volta install ember-cli 3.9.1
    volta install typescript 3
    ```

-   **Project toolchain:**

    ```sh
    volta pin ember-cli 3.9.1
    volta pin typescript 3
    ```

-   **Tool retrieval:**

    ```sh
    volta fetch ember-cli 3.9.1
    volta fetch typescript 3
    ```

This is a surprising difference for users coming from `npm` or `yarn`. Switching over to use the same invocation as the existing tools will make adoption smoother and less surprising.

# Pedagogy
[pedagogy]: #pedagogy

Installation, fetching, and pinning function much as they do today, with two changes: the tool + version combination is now a single string, in the format `tool[@<version>]`; and users may fetch, install, or pin multiple tools at once.

## New tool + version string

The tool + version combination is now a single string, in the format `tool[@<version>]`.

Users may supply a tool with no version string:

```sh
volta install node
```

This will use the current default for the specified tool—currently the latest version available for any package, and the LTS distribution for Node.

Users may supply a tool with a <i>version</i> or a <i>version spec</i>, using an `@` to separate the tool name from the version or version spec.

Valid <i>versions</i> are any of the following formats:

-   `node@10`
-   `node@10.5`
-   `node@10.5.2`

Valid <i>version specs</i> are any valid SemVer spec, as already implemented in Volta. As a representative (not exhaustive) sample:

-   `node@10.5.*`
-   `node@^10.5`
-   `node@~10.5`
-   `node@>=10`
-   `node@<10`
-   `"node@8 || 9"` (an odd invocation, but one our current implementation supports)

Finally, users may supply a tool with either of the two special version names Volta currently supports, `lts` and `latest`:

-   `node@lts`
-   `node@latest`

This RFC does *not* propose supporting arbitrary tags such as `next`, as there is further design work to be done there; but it does not preclude support for that in the future.

## Multiple tool + version arguments

The invocations now allow multiple tool + version combinations, each of the format specified above. For example, a user may simultaneously pin a Node and a Yarn version (with any combination of the version specifier, or none at all):

```sh
volta install node@10.5 yarn
```

## Explanation for Users

### Current Node users

There are two broad sets of current Node users: those not using a Node version management tool, and those using an existing tool like `nvm`, `nodenv`, or `nvm-windows`.

Current Node users who are not using a Node version management tool should find this a simplification: it matches what npm, npx, and Yarn do.

Current Node users using existing tools will find this to be different from their experience:

-   `nvm` uses `nvm install <version>`
-   `nodenv` uses `nodenv install <version>`
-   `nvm-windows` uses `nvm install <version> [arch]` (where `[arch]` allows users to specify 32-bit or 64-bit builds)

Although we are breaking with these tools, all of them are responsible to manage *only* the Node version, *not* other tools. Volta's purview extends to other tools and therefore overlaps more with the behavior expected from invoking `npm`, `npx`, or `yarn`.

### New Node users

New Node users may have a wide variety of different expectations for adding a version of a tool:

-   RubyGems: `gem install <gem name> [-v|--version <version>]`

-   Bundler: `bundle add <gem name> [--version=<version>]`

-   Nuget: `nuget install <package> [-Version <version>]`

-   Paket: `paket add <package> [--version <version spec>]`

-   `dotnet`: `dotnet tool install <PACKAGE_NAME> <-g|--global> [--version]`

-   Cargo: `cargo install crate [--version <version>]`

-   `cargo-edit`: `cargo add <package>[@<version>]...`

Of these, only `cargo-edit` allows adding multiple packages simultaneously; it does so with the same interface as `npm` and `yarn` and proposed here.

# Details
[details]: #details

## Installing, fetching, or pinning a single tool

Managing a tool with its default version is unchanged.

| Original                  | Proposed                  |
| ------------------------- | ------------------------- |
| `volta install node`     | `volta install node`     |
| `volta pin yarn`         | `volta pin yarn`         |
| `volta fetch typescript` | `volta fetch typescript` |

Managing a tool at a specific version, at any level of granularity, and with either a specific version or a version spec, now uses `@` to separate the tool name from its version/version spec.

| Original                        | Proposed                        |
| ------------------------------- | ------------------------------- |
| `volta pin node 10`            | `volta pin node@10`            |
| `volta fetch node 10.5`        | `volta fetch node@10.5`        |
| `volta install node 10.5.0`    | `volta install node@10.5.0`    |
| `volta install node ^10`       | `volta install node@^10`       |
| `volta install node ~10.5`     | `volta install node@~10.5`     |
| `volta install node >=10`      | `volta install node@>=10`      |
| `volta install node <10`       | `volta install node@<10`       |
| `volta install node "8 || 10"` | `volta install "node@8 || 10"` |

This also works for shorthands (currently `latest`, conceivably also tags like `next` in the future):

| Original                 | Proposed                  | Status      |
| ------------------------ | ------------------------ | ------------ |
| `volta pin node latest` | `volta pin node@latest` | current      |
| `volta pin node lts`    | `volta pin node@lts`    | current      |
| `volta pin node next`   | `volta pin node@next`   | hypothetical |

## Installing, fetching, or pinning multiple tools

Previously, installing multiple tools required invoking Volta multiple times. Now, it may be invoked with multiple tools at once.

- **Original:**

    ```ts
    volta install create-react-app latest
    volta install typescript 3.4
    ```

- **Updated:**

    ```
    volta install create-react-app@latest typescript@3.4
    ```

There are no impacts to features other than fetching, installing, and pinning.

## Implementation changes

The command line implementation will need to be updated, with the install, fetch, and pin commands now receiving a list of tool specs.

The command line parsing will need to be updated to accept variadic arguments of tool-with-optional-versions, using Clap's [`Arg::multiple`](https://docs.rs/clap/2.32.0/clap/struct.Arg.html#method.multiple) argument, with a custom parser. E.g., for `install`, we might have something like this

```rust
#[derive(StructOpt)]
pub(crate) struct Install {
    /// The tool to install, e.g. `node` or `npm` or `yarn`
    #[structopt(multiple = true), parse(try_from_str = "ToolSpec::try_from_str")]
    tools: Vec<ToolSpec>,
}
```

Here we assume that the `ToolSpec` will add a `try_from_str` or `try_from_os_str` implementation, with a useful error type—here hand-waved as `String`, but it could be as complex as we need as long as the error type has an implementation of `Into<String>`:

```rust
impl ToolSpec {
  fn try_from_str(string: &str) -> Result<ToolSpec, String> {
    // split apart the tool name and its version
  }
}
```

This is a relatively simple implementation. The only complication is the existence of Node namespaces: `@namespace/package`. This means parsing the version is not as simple as splitting on `"@"`. However, it should still be straightforward using the combination of a regular expression and the existing semver tooling. The use of `@` to demarcate a namespace is restricted to the first character, which means the problem is small. In any case, we will treat npm and Yarn's handling as reference implementations.

The `ToolSpec` change will make the format `tool[@version]` work; updating the commands to take a `Vec<ToolSpec>` will support passing multiple arguments.

# Critique
[critique]: #critique

This introduces breaking changes—something which should not be done lightly! It does so under the assumption that fetching, installing, or pinning multiple tools simultaneously is a relatively common operation (relative to the frequency of running those operations at all). This assumption may be wrong.

It also introduces a move away from users’ existing habits with `nvm` and `nodenv` and similar tools. This is thus an increase in the learning/switching costs for the tool.

*Not* doing this is not likely to be a blocker for wider adoption. The “problem” identified here represents a papercut, not a major pain point.

## Alternatives

-   Support *both* invocation styles. This may *just* be technically feasible, allowing users to pass any of the following invocations successfully:

    ```sh
    volta pin create-react-app
    volta pin create-react-app@3.0
    volta pin create-react-app 3.0
    volta pin create-react-app typescript
    volta pin create-react-app@3.0 typescript
    volta pin create-react-app typescript@3.4
    volta pin create-react-app@3.0 typescript@3.4
    ```

    However, this introduces substantial additional complexity for the implementation, and ambiguity around certain invocations, e.g. for tagged versions/special names. For example, how should the following invocation be resolved?
    
    ```sh
    volta pin create-react-app latest
    ```

    Does it refer to the latest version of `create-react-app`, or the default version of `create-react-app` and [the `latest` package](https://www.npmjs.com/package/latest)? For all non-SemVer-string parses, this will be impossible to resolve.

    The proposed design here is well-proven with other tools which install more than simply a single tool-and-version pair, e.g. `npm`, `yarn`, and `cargo-edit`, and it does not fall prey to these problems of indeterminacy.

-   Leave things as they are. This has the upside of matching other Node version managers, all of which use a close variant on our current syntax. However, none of those tools support installing anything but Node, so the comparison is less telling than it might seem.

# Unresolved questions
[unresolved]: #unresolved-questions

-   How do we handle errors when installing multiple things?

    -   Do we keep going and try to install as much as we can, or fail on the first error?
    -   If we do attempt to continue and multiple installs fail, how do we present those errors to the user?

## Future work

Introducing this feature will *suggest* to users that they can use tags, e.g. `tool@next`, a feature we do not currently support. We should plan that as useful later work, as it requires further design around the difference between Node and other tools, since other tools are npm packages with tags, and Node is not. Moreover, if or when we implement that functionality the version-specifying approach proposed here will get it "for free".
