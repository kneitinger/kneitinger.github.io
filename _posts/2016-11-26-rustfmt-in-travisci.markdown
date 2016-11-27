---
layout: post
title:  "Ensuring Beautiful Commits with rustfmt and Travis-CI"
subtitle: "With only a few small tweaks, your builds can be as picky as you!"
date:   2016-11-26
author: Kyle Kneitinger
group: tutorials
---

#  When Style is Standardized, Style Can Be Standard
The [rust-lang-nursery GitHub
organization](https://github.com/rust-lang-nursery) is a fantastic group of
folks building tools for both working in, and working with, the Rust language.
One of these tools, [rustfmt](https://github.com/rust-lang-nursery/rustfmt), is
quite helpful for maintainting a consistent code style throughout a project.

`rustfmt` is pretty easy to work with.  It's default behavior, when executed on
a file, is to check it for anything that violates the style guide, and if
anything needs to be changed, backup the file with a `.bk` suffix and silently
replace the original with the new formatting. If passed a valid rust file, it
returns 0 regardless of any changes being made. `rustfmt` only operates on a
single file, so if the current directory is a `cargo` project, a handy `cargo
fmt` wrapper is provided that checks the entire project.

There are a few nice ways to integrate `rustfmt` into your workflow, such as
[vim plugins](https://github.com/rust-lang/rust.vim#formatting-with-rustfmt),
and git commit hooks.  Unfortunately, these depend on the individual developer
to use them, and thus do not enforce any project-wide policy.

# Enter Travis
A lot of the themes brought up in the previous paragraph are pretty reminiscent
of testing: consistency when used, but largely reliant on the developer to use
them. Since Travis-CI was
already set up to run our tests and report upon failures, why not leverage that build process to handle
formatting?

The default rust `.travis.yml` provided by Travis CI is pretty straightforward:
{% highlight yaml %}
language: rust
rust:
  - stable
  - beta
  - nightly
matrix:
  allow_failures:
    - rust: nightly
{% endhighlight %}

It relies on `cargo` to run `cargo build --verbose` and `cargo test
--verbose` during the implict "script" phase of the build.
With just a few slight modifications, we can have format checking be a part of
this process.

# Installing the Tool

Travis CI can has an optional "install" step (see [Travis-CI: The Build
Lifecycle](https://docs.travis-ci.com/user/customizing-the-build/#The-Build-Lifecycle))
where we can have `cargo` install `rustfmt` and then add it to our path. This step can take a very long time to finish due to compilation of the `env_logger` crate.  Thankfully Travis CI provides a mechanism for caching dependencies with the `cache` key. Putting what we have so far together, we get:

{% highlight yaml %}
language: rust

cache: cargo

rust:
  - stable
  - beta
  - nightly

matrix:
  allow_failures:
    - rust: nightly

install:
  - (cargo install rustfmt || true)
  - PATH=$PATH:/home/travis/.cargo/bin
{% endhighlight %}

Recall how I said that any non-zero return status breaks the build?  Well the
`cargo install` command returns non-zero if the desired package is already
installed (which means caching worked!). In order to get around this,
we massage it into returning true upon "failure" and thus continuing the build.

# Checking the Style

Now that our build context has `rustfmt`, we can have it run along with the
builds and test. It would be nice if we could have Travis-CI automatically and politely run `rustfmt` and fix any format-offensive code, but that would have to rewrite git history and is out of the question.  The best we can do is break the build upon a push or pull request, so that the devloper knows to run the format command and recommit.

Since any command with a non-zero return status breaks the build, we just need `cargo fmt` to break at some point.
Again, the default behavior of `rustfmt`, even when wrapped
into `cargo`, is to silently backup and reformat and offending code. We need a
way to detect when code is malformatted.  If we check the usage with `rustfmt -h`, we see the `--write-mode [replace|overwrite|display|diff|coverage|checkstyle]` option.
The default setting is 'replace', but after trying them out, it seems the most
helpful mode for this purpose is 'diff'.  It provides two things that work well
in this context: a non-zero return code upon code that doesn't meet the style
guide, and output that makes it clear why the build broke. Since we're using the project-aware `cargo fmt` wrapper, we cannot use
`rustfmt` flags directly until we insert a `--` argument.

At this point, we *could* just put the `cargo fmt` line in the "install step",
it would certainly work as desired.  However, for the sake of clarity and
further modularity, let's make the "script" step explicit now.  This means
inserting the `cargo fmt` command as well as the default `build` and `test`
commands as elements of the "script" key:

{% highlight yaml %}
language: rust

cache: cargo

rust:
  - stable
  - beta
  - nightly

matrix:
  allow_failures:
    - rust: nightly

install:
  - (cargo install rustfmt || true)
  - PATH=$PATH:/home/travis/.cargo/bin

script:
  - cargo fmt -- --write-mode=diff
  - cargo build --verbose
  - cargo test --verbose
{% endhighlight %}

And there you have it! I hope this helps you make the world a cleaner, more stylish place! If you have any feedback or questions, feel free to
comment below or reach out on twitter.
