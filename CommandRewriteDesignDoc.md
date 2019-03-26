# WPILib Command Framework Rewrite
Design document

The WPILib Command-based framework has been a staple of FRC Java and C++ robot code development for years, offering teams a relatively clean and sensible API for factoring complex robot code into simple, encapsulated chunks that interact in clear, comprehensible ways.  In addition to helping teams avoid the often substantial overhead of rolling their own state machine logic, it also serves as an important pedagogical tool for teaching teams about encapsulation, control flow, and design patterns.

However, the libraries used in the framework are now more than a decade old.  Many of the particular choices in their implementation are obsolete, and others are messy, difficult-to-maintain, and/or confusingly-conceived.  Given the confusing nature of much of the implementation, bringing the API up to modern standards is a difficult task, and the libraries have ended up in a "stasis" of sorts.

To document will describe the goals, design choices, and implementation of a ground-up rewrite of the Command-based libraries aimed at alleviating these issues.


## Why rewrite the framework? (Problem Summary)

The problems with the current command framework fall generally into three major categories:

1.  Poor readability/maintainability.
2.  Poor encapsulation/separation of responsibilities.
3.  Unnecessarily restrictive API design.
4.  Clutter

These three problems are interrelated, but I will summarize each in turn.

### Problem 1: The Code Is a Mess

It is well-known that, despite the relatively simple API, the underlying code of the Command-based libraries is a mess.  The `Scheduler` class is particularly notorious - the code is poorly-documented, difficult to follow, and uses a large array of sketchy custom implementations of features that have since become standard Java utilities.  Both `Command` and `CommandGroup` are quite lengthy and difficult-to-follow, with the comments warning about various potential violations-of-assumptions re: the scheduling state of the commands but without any clear guide to what those actual assumptions *are*.  `Subsystem` contains quite a bit of state despite not really having any reason to do so.  These issues are aggravated by...

### Problem 2: It Is Not Clear What Each Class Is Supposed to Be Responsible For

Despite its name, `Scheduler` does not really handle scheduling, except for the parts of it that it does.  `Command` has an API that suggests it functions merely as a state machine abstraction of user code, but contains large amounts of scheduling state.  `Subsystem` is responsible for managing its own default command, except for the parts that are handled in `Scheduler`.  While the names of the objects in the library imply a fairly reasonable division of responsibilities, this is not followed in practice, and the resulting control flow is a convoluted mess.

In particular, `Scheduler`'s status as a singleton has been quite badly abused, resulting in unpredictable and very rigid coupling to distant parts of the library.  The seemingly-arbitrary shunting of actual scheduling logic into `Command` makes it difficult to ascertain where in the code one would need to go to access a given piece of information about the robot state.  The simultaneous handling of `Subsystem` requirements by `Scheduler` and `CommandGroup` makes it unclear how one is supposed to check the current status of a `Subsystem` - as a result, knowledge of running commands has been needlessly duplicated in `Subsystem`, introducing both confusion and potential failure-states.

### Problem 3: The Code Is Not Flexible In Places Where It Could Be

Because the division-of-responsibilities is so ambiguous, large amounts of state have leaked into almost every class in the library.  As a result, almost nothing is an interface, while almost everything is an abstract class.  As Java does not support multiple inheritance, this is a problem - mandating that `Command`s and `Subsystem`s inherit from a library class places a fairly heavy burden on the potential inheritance structure of team code.  Given that teams only really substantially interact with these classes through method overrides, this makes little sense.

Additionally, `CommandGroup` is a confusing and difficult-to-use class whose structure serves to obscure the potential power of its abstraction.  The `addParallel()` and `addSequential()` API is unintuitive, and unnecessary - structures that can be built within a `CommandGroup` through these methods can be more-cleanly built through multiple `CommandGroup`s combined through recursive composition.

Finally, the API has been designed almost exclusively for use through the subclassing of `Command` and `CommandGroup`.  Additions circumventing this pattern do exist (e.g. `InstantCommand`), but much Command-based code is needlessly verbose.  The API's classes are, by-and-large, not usable as-is; they are usable only as foundations for user implementations.  This makes it unduly difficult for new teams to accomplish simple tasks, such as configuring working drive code, in comparison to a simple state machine in `Robot.java`.  As the benefits of the Command-based framework are not seen until teams begin to attempt more-complicated robot functionality, many teams who are initially discouraged by this end up without those benefits when they are needed; and even teams that stick with it are saddled with unnecessary verbosity for simple tasks.

### Problem 4: The API Is Really, Really Cluttered

Because of deeper issues with separation-of-concerns between the Command-based libraries and the `Sendable` implementation, many `Command`s have long lists of overloaded constructors and other cruft that make the code difficult to navigate and provide little useful functionality to the users.  The `PIDCommand` class is a particularly egregious example.

Overall, the problems with the library run deep enough that an attempt to modify the existing structure would likely be futile.  Large, breaking changes are needed to address the concerns raised above.

## Overall Design of the Rewrite

The rewritten Command-based framework seeks to rectify the above problems.  A broad overview of the changes introduced in the rewrite is as follows:

1.  `Command` and `Subsystem` have been refactored to be interfaces instead of abstract classes.  This places much less imposition on user code, and also prevents scheduling responsibilities from leaking into them.  `Subsystem` is no longer required to have any knowledge of its default `Command`.
2.  `Scheduler` (now named `CommandScheduler`) is solely responsible for the scheduling of commands (except as occurs through composition) and the handling of requirements.  All relevant scheduling state lives only in scheduler (as `Command` and `Subsystem` are now interfaces, and thus no no longer stateful), though some is further encapsulated in private `CommandState` objects.
3.  `CommandGroup` has been split into several more-basic classes: `SequentialCommandGroup`, `ParallelCommandGroup`, `ParallelRaceGroup`, and `ParallelDictatorGroup`.  These classes implement the `Command` interface, and are thus composeable - the resulting composition-based API is both more-powerful and much cleaner than the previous `CommandGroup` class.
4.  To facilitate composition of commands, `Command` has been given a large number of defaulted "decorator" methods which wrap the command with a decorated functionality, usually by means of one of the existing `CommandGroup` classes.
5.  A large number of useable-out-of-the-box, lambda-based `Command` classes have been included.  In conjuction with the decorator methods mentioned above, these allow teams a simple, concise way to define powerful robot behaviors without writing any new classes.
6.  `Sendable` implementations have been moved into abstract base classes, which are used as a base for the provided `Command` implementations and `CommandGroup` classes.  These are purely provided for user convenience, and users may disregard them and roll their own `Sendable` functionality if desired (though it is not really possible to decouple this from our pre-made `Command` implementations).

### In-Depth Examples (How Does it Work?)

#### Command

As mentioned earlier, `Command` has been refactored to be an interface.  This has changed surprisingly little about how users interact with it, as its user-facing function in the original library was mostly that of an interface to begin with.

##### Basic state machine logic

```java
default void initialize() {}
default void execute() {}
default void end(boolean interrupted) {}
default boolean isFinished() { return true; }
default boolean runsWhenDisabled() { return false; }
```

These work exactly as before, with the exception of `end()`, which has been modified to fill the roll of both `end()` and `interrupted()` in the previous library (the method param indicates whether the command finished normally, or was interrupted).  For the rest of the document, when we say a command is "interrupted," this means `end(true)` is called; when we say "finished," we mean `end(false)` is called.

The methods are defaulted for convenience, so that overriding is not necessary if one does not wish to use one of them.  The only difference is that to specify run-when-disabled behavior, one must now override the `runsWhenDisabled()` method instead of calling a `setRunsWhenDisabled()` method (which is not feasible, as `Command` is no longer stateful).

##### Handling requirements

```java
Set<Subsystem> getRequirements();
default boolean hasRequirement(Subsystem requirement)
```

`Command`s declare their `Subsystem` requirements by overriding the `getRequirements` method.  Users are advised to include their requirements as a field in their `Command` implementation, to avoid needless reallocation when this method is called.  The provided `SendableCommandBase` class does this for users, as well as providing a simpler `addRequirements()`-based API.  This method is not defaulted, to ensure users do not forget about it.

The `hasRequirement()` method returns whether the given subsystem is a requirement.

##### Convenience wrappers on CommandScheduler methods

```java
default void schedule(boolean interruptible)
default void schedule() { schedule(true); }
default void cancel()
default boolean isScheduled()
```

These methods simply wrap the `CommandScheduler` methods for convenience, and are self-explanatory.

##### Composeable decorators

```java
default Command withTimeout(double seconds)
default Command interruptOn(BooleanSupplier condition)
default Command whenFinished(Runnable toRun)
default Command beforeStarting(Runnable toRun)
default Command andThen(Command... next)
default Command dictating(Command... parallel)
default Command alongWith(Command... parallel)
default Command raceWith(Command... parallel)
```

These methods compose the given command within a CommandGroup for added functionality.  They are extremely concise and powerful, and pretty self-explanatory.  The three parallel methods correspond with `ParallelDictatorGroup`, `ParallelCommandGroup`, and `ParallelRaceGroup`, respectively.  With these, one can create quite-substantial structures in only a small bit of code:

```java
button.whenPressed(foo.raceWith(bar.andThen(baz.withTimeout(5))));
```

#### Subsystem

The `Subsystem` class has, like `Command`, been refactored to be an interface rather than an abstract class.  It is also no longer responsible for keeping track of its own default `Command` - that responsibility now properly belongs to `CommandScheduler`.  As a result, there is really not much left in it, which is a good thing.

Subsystems should be registered with the `CommandScheduler` through the `registerSubsystem` method.

##### Periodic

```java
default void periodic() {}
```

A method that gets called once per run of the `CommandScheduler` on all registered `Subsystem`s.  Unchanged from the previous command framework.

##### Default commands

```java
default void setDefaultCommand(Command defaultCommand)
default Command getDefaultCommand()
```

While `Subsystem` is no longer responsible for managing its default `Command`, it does provide these two convenience wrappers on the relevant `CommandScheduler` methods.

##### Querying command state

```java
default Command getCurrentCommand()
```

A convenience wrapper on the `CommandScheduler` method that returns whatever command is currently scheduled that requires this subsystem.

#### Command Groups

Command groups allow users to compose individual commands into more-complicated commands.  The `CommandScheduler` does *not* see command group internals; every command group appears to the scheduler as if it were a single command.  This maintains encapsulation, and allows compositions of arbitrary complexity without any modification to the `CommandScheduler`, resulting in a much easier to read and easier to maintain library.

There are four basic types of Command groups:

```java
SequentialCommandGroup(Command... commands)
ParallelCommandGroup(Command... commands)
ParallelRaceGroup(Command... commands)
ParallelDictatorGroup(Command dictator, Command... commands)
```
The first two are self-explanatory.  

`ParallelRaceGeoup` is a parallel command group that terminates when *any one* of the included commands finishes.  All other commands are interrupted at that point.

`ParallelDictatorGroup` is a parallel command group that terminates when a specified one of the included commands (the "dictator") finishes.  All other commands that are still running at that point are interrupted.

As in the previous library, all command groups require the union of the requirements of their components.  This may seem cumbersome, but teams that wish to circumvent it are free to neglect to declare requirements, or else use `ScheduleCommand` to fork off independently from a command group.  For most use-cases, this is the most-intuitive behavior, as requirements are only checked upon the initial scheduling of a command, and it is by far the easiest to implement and to maintain.

Also as in the previous library, command instances may not belong to more than one `CommandGroup`, and may not be independently scheduled if they are part of a `CommandGroup`.  This prevents inconsistent internal `Command` state.  This is accomplished by means of a weak static list of "allocated" `Command`s in the `CommandGroupBase` class.  Static methods are available for users to clear `Command`s from this list, if desired, but it is strongly warned against in the documentation as there are only a couple of legitimate use-cases for it.

#### CommandScheduler

The `CommandScheduler` serves the function of the `Scheduler` in the previous command framework.  It has been entirely rewritten to use modern Java collections APIs, and to have a much simpler and more readable control flow than the previous `Scheduler`.  The `CommandScheduler` remains a singleton, to allow global access from across the robot program - this is important for facilitating convenience methods in both the `Command` and `Subsystem` interfaces, as well as not requiring users to inject a `Scheduler` across their entire codebase, which is tedious and provides little real benefit.  However, the new library does a much better job of avoiding rigid coupling to the `Scheduler` object, even though it is globally-acessible.

The `CommandScheduler` API is very simple:

##### Scheduling commands

```java
void scheduleCommand(Command command, boolean interruptible);
void scheduleCommands(boolean interruptible, Command... commands);
```

The `interruptible` boolean sets if the command is permitted to be interrupted by a later-scheduled command that needs one of its required `Subsystems` (previously, this was set permanently for a given command - the new implementation is more flexible).  When a `Command` is scheduled, the `CommandScheduler` first checks to see if its requirements are free.  If they are, the command is scheduled, its `initialize()` method is run, and its requirements are added to the list of currently-used requirements.  If its requirements are not all free, it is checked if all of the commands using requirements that it needs have been scheduled as `interruptible`; if they have, all of these commands are interrupted, their requirements are freed, and the command is scheduled as above.  If not, nothing is done.  The vararg version of the method simply runs the process repeatedly for multiple commands.

##### Registering subsystems/default commands

```java
void registerSubsystem(Subsystem... subsystem);
void unregisterSubsystem(Subsystem... subsystem);
void setDefaultCommand(Subsystem subsystem, Command defaultCommand);
```

These methods register subsystems so that their periodic methods will be called when the scheduler runs, and their default commands will be scheduled when they are not currently being required by other commands.  Setting a default command automatically registers the subsystem.

##### Running commands/subsystems

```java
void run();
```

This functions almost exactly as before, except the actual responsibility of tracking the scheduling state of the commands now does belong to the `CommandScheduler`.

##### Canceling commands

```java
void cancelCommand(Command... commands);
void cancelAll();
```

Forceably ends the given commands, if they are currently scheduled.  The commands will have `end(true)` called, rather than `end(false)` methods.  Note that a command can be cancelled ever if it is not scheduled as `interruptible` - the `interruptible` tag *only* determines if the command can be interrupted by *another command* through its requirements.

##### Querying command state

```java
boolean isScheduled(Command command);
double timeSinceScheduled(Command command);
Command requiring(Subsystem subsystem);
Command getDefaultCommand(Subsystem subsystem);
```

These methods are fairly self-explanatory, and provide ways for the user to get information about the state of various `Command`s or `Subsystem`s.  They are often wrapped through convenience methods in the `Command` and `Subsystem` interfaces.  Note that `isScheduled()` and `timeSinceScheduled()` do not provide information about Commands that have been composed within CommandGroups - the scheduler does not see CommandGroup internals.

##### Adding additional actions to perform on execution of command subroutines

````java
void onCommandInitialize(Consumer<Command> action);
void onCommandExecute(Consumer<Command> action);
void onCommandInterrupt(Consumer<Command> action);
void onCommandFinish(Consumer<Command> action);
````

These provide a simple way to add a task for the scheduler to perform whenever a command is initialized, executed, interrupted, or finished normally.  For example, one could insert a logging call to mark the event.

#### Sendable base classes

The new Command library includes `Sendable` base classes for both `Command` and `Subsystem` that teams may (but are in no sense required) to use.  No constructor parameter for name is provided, to avoid inducing similar overloaded constructor clutter in subclasses as is seen in the original command-based library.  Teams can call the `setName()` method prior to sending if they wish to set a name.

##### SendableCommandBase

A base class for `Command`s that implements `Sendable`.  All provided `Command` implementations are built from this, as are the `CommandGroup`s.  Also provides a field for requirements, as is advised for implementations of `Command`, and a clean vararg `addRequirements()` method.

##### SendableSubsystemBase

A base class for `Subsystem`s that implements `Sendable`.

#### Included command implementations

The library comes with a very large array of `Command` implementations built from `SendableCommandBase`.  There are too many of these to be worth summarizing here, but they are quite well-documented and easy to grasp.  The majority of these are heavily functionalized, allowing use "out of the box" via. lambdas rather than requiring users to subclass them.  For example, a default drive command might be implemented in the following way:

```java
driveSubsystem.setDefaultCommand(new RunCommand(() -> driveSubsystem.tankDrive(leftJoystick.get(), rightJoystick.get()), driveSubsystem));
```

This is simple, straightforward, and requires no subclassing.  The prebuilt commands have been kept as general as possible, to avoid their number growing beyond what can be reasonably maintained, and to keep their dependencies to a minimum.

#### PIDCommand and PIDSubsystem

These convenience wrappers have been kept from the previous library in spirit, but have been greatly updated in implementation.  For starters, the new PIDController (courtesy of @calcmogul) is used, which is able to be run synchronously from the main loop.  Accordingly, `PIDCommand` and `PIDSubsystem` have been split into asynchronous and synchronous versions.  Thread safety warnings have been added to the asynchronous versions.

Both `PIDCommand`s now take lambdas for setpoint, process variable, and output.  On the other hand, `PIDSubsystem` only takes a lambda for process variable measurement (and this will likely change when future updates to the PIDController move that lambda out of the controller object itself), as "out-of-the-box" use makes much less sense for a `Subsystem` than for a `Command` (as users will want to write their own class for the `Subsystem` anyway, in order to encapsulate resources within it).

#### Trigger and Button

The `Trigger` class has (and its subclasses) hardly been modified for the new library, and works nearly exactly as it did before.  However, some new additions are worth noting:

##### Interruptible param

All `Trigger` and `Button` classes have had an `interruptible` boolean added to their binding methods, to match the new feature of the `CommandScheduler`.  Overloaded methods have been provided that default this parameter to `true`.

##### whileActiveOnce vs whileActiveContinuous

The `whileActive()` method previously re-scheduled the command continuously while the trigger was active.  This has now been named `whileActiveContinuous()`, and a `whileActiveOnce()` method has been added that cancels the command when the trigger becomes inactive, but does not re-start it if it finishes while the trigger is still active.  The corresponding `Button` method names are `whileHeld()` and `whenHeld()`.

##### Runnable bindings

The `whenActive`, `whileActiveContinuous`, and `whenInactive` bindings (and their corresponding `Button` equivalents) have been given the ability to take a `Runnable`, which is wrapped in an `InstantCommand` automatically.  This allows the use of the edge-finding logic for tasks that the user does not wish to write a command for.

##### Trigger composition

```java
Trigger and(Trigger trigger);
Trigger or(Trigger trigger);
Trigger negate();
```

These allow `Trigger`s to be composed prior to button binding, a la:

```java
button1.and(button2).whenActive(...);
```

