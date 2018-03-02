# What not to do

Sometimes I feel like the only things I know about somewhere are _negative cases_. Here are some of them.


## Building, continuous integration, etc

### Don't configure your build in your CI server

Products like TeamCity and VSTS have wonderful UIs for creating and editing build pipelines. You should not use them.

#### Instead: build scripts should be checked in artifacts

* They follow the code. Changing the build on a branch doesn't affect other branches, and will follow the code when it's merged.
* Changing the build doesn't break the ability to check out old code and build it. This makes tools like `git bisect` much more useful.
* You don't need to back up your CI server builds.
* The build is portable across any CI target (internal, public Travis/Appveyor, etc).

## Code-level

### Don't start hidden parallel background operations

This leads to hard-to-test code and an invasion of `Thread.Sleep`s.

#### Instead: always give the caller some way to wait upon completion

Even if you don't use it in your production code (and let it run off in the background), return a `Task` so that test code can avoid running into timing issues.
