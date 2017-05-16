# (Poor man’s) model-based testing with F# & FsCheck

*This is a blog version of a talk that I did ~2 years ago in Auckland. Nothing in it is original or unknown, but there aren’t that many examples of doing this aside from [in the FsCheck documentation](https://fscheck.github.io/FsCheck/StatefulTesting.html).*

### What is FsCheck?

FsCheck is a library in the spirit of Haskell’s [QuickCheck](https://en.wikipedia.org/wiki/QuickCheck), which allows you to perform property-based testing. The main feature that it provides is the ability to generate arbitrary instances of any supported data type, and the ability to define your own generators.

We can leverage this ability to perform model-based testing in a simple manner.

Let’s run through how we can use this to test a simple system. All the code in this post is going to be F#, but the system could equally well be described in C# and tested from F#. If you copy and paste the code into an F# file, it should work (you’ll need to install the `FsCheck.Xunit` and `xunit` packages, and probably `xunit.runner.visualstudio` to run the tests).

### The system to test

In our basic system, we store users. We can `Add`, `Get` (possibly failing), or `Delete` a user. This is described by the following interface:

<script src="https://gist.github.com/Porges/c7a6c6abf5e22b1e65a7445e8ff47ce0.js?file=00-system.fs"></script>

Next we have to implement this system.

We'll use an in-memory dictionary, where we map `UserID`s to the actual `User` objects:

<script src="https://gist.github.com/Porges/c7a6c6abf5e22b1e65a7445e8ff47ce0.js?file=01-implementation.fs"></script>

*Unfortunately*, on line 20 we’ve introduced a critical bug. If the user ID contains “`*`” then we accidentally remove all the users in the system. Oops. 🤦

Later on we will see if our test will find this bug.

### Modelling the system

In order to check our system, we must develop a model—a simplified version of the system that we can verify more easily. The model doesn’t need to replicate every piece of behaviour of the full system, it might only replicate one aspect that we care about.

Here we will use a version of the system that only cares about *how many* users there are. In order to track this it only needs to store the set of valid IDs:
<script src="https://gist.github.com/Porges/c7a6c6abf5e22b1e65a7445e8ff47ce0.js?file=04-model.fs"></script>

`Add` adds an ID to the set (unless it already exists), `Delete` removes an ID (if it exists), and `Get` does nothing.

We also provide a `Verify` method that checks the model against the real system. (In real life, this could use HTTP calls or some other method.)

### Applying operations

Now, in order to test our system, we need a way to describe the actions we can perform against it. We do this by *reifying* the operations we can perform as a data type:

<script src="https://gist.github.com/Porges/c7a6c6abf5e22b1e65a7445e8ff47ce0.js?file=02-operations.fs"></script>

See that that this aligns exactly with the `IUserOperations` interface shown in the first block of code above. For each method on the interface, we have a corresponding case in the discriminated union, and the data needed to call the method is also contained in the case.

Next, we need to be able to perform these actions against both the model and the real system. To do this we simply translate from each case to the corresponding method on the interface:

<script src="https://gist.github.com/Porges/c7a6c6abf5e22b1e65a7445e8ff47ce0.js?file=03-apply-operations.fs"></script>

### Testing the system

Now we get to the magic part! First up, we have a little boilerplate to tell FsCheck that we don’t want any `null` strings generated, as this would just complicate the example:

<script src="https://gist.github.com/Porges/c7a6c6abf5e22b1e65a7445e8ff47ce0.js?file=05-tests-boilerplate.fs"></script>

And now our test. We take in a list of operations (which FsCheck will automatically generate for us), we apply all the operations to both models, and then we verify the system against the model:

<script src="https://gist.github.com/Porges/c7a6c6abf5e22b1e65a7445e8ff47ce0.js?file=06-tests.fs"></script>

This may seem like less code than expected! FsCheck can automatically build us a list of `Operation`s because it knows how to make `Operation` (since it knows how to make each case in turn,starting from `string`s and building up). Recursion is great!

### Running the test and “shrinking”

If you try to run the test, you should receive a result along these lines (the exact details will differ):

![](/content/images/2017/02/shrunk.PNG)

Our test failed (that’s a good thing!) See the second-to-last `Delete` operation in the input? It is deleting `UserId "^*VJc6'/I^x"`, which triggers the bug we have in our system for user IDs containing “`*`”.

The input it’s showing us is quite gnarly as FsCheck has generated user IDs containing newlines and other nasty characters, but the highlighted part at the bottom is what FsCheck was able to shrink the failing input to.

It can do this because, as well as knowing how to *generate* random data, FsCheck knows how to *shrink* random data. It was able to repeatedly shrink the input until it found this minimal case, containing only two operations:

1. Add a user (note that it doesn’t seem to matter what the details are, as all the attributes shrank away to nothing–if the details mattered, then it wouldn’t have shrunk this much)
2. Delete the user with ID “`*`”

And indeed, this sequence exactly replicates our bug! 🎉 

The combination of randomly generating inputs and then being able to shrink them when failures are found is *very* powerful.

### The recipe

A simple list for success:

1. model your system-under-test as an interface
2. create a model of your system (or multiple models, each testing one aspect) that also implements the interface
3. create a type that reifies each operation as a member of a discriminated union
4. create a test that applies a list of operations to the model and system and validates the system
5. 💅 relax and let your computer do the work
