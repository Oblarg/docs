# WPILib Command Framework Rewrite
Design document

The WPILib Command-based framework has been a staple of Java and C++ robot code development for years, offering teams a relatively clean and sensible API for factoring complex robot code into simple, encapsulated chunks that interact in clear, comprehensible ways.  In addition to helping teams avoid the often substantial overhead of rolling their own state machine logic, it also serves as an important pedagogical tool for teaching teams concepts about encapsulation, control flow, and design patterns.

However, the libraries used in the framework are now more than a decade old.  Many of the particular choices in their implementation are obsolete, and others are messy, difficult-to-maintain, and/or confusingly-conceived.  Given the confusing nature of much of the implementation, bringing the API up to modern standards is a difficult task, and the libraries have ended up in a "stasis" of sorts.

To document will describe the goals, design choices, and implementation of a ground-up rewrite of the Command-based libraries aimed at alleviating these issues.


## Why rewrite the framework? (Problem Summary)

The problems with the current command framework fall generally into three major categories:

1.  Poor readability/maintainability.
2.  Poor encapsulation/separation of responsibilities.
3.  Unnecessarily restrictive API design.

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

#### The CommandScheduler

The central class of the Command-based rewrite is the `CommandScheduler`.  The `CommandScheduler` serves the function of the `Scheduler` in the previous command framework.  It has been entirely rewritten to use modern Java collections APIs, and to have a much simpler and more readable control flow than the previous `Scheduler`.  The `CommandScheduler` remains a singleton, to allow global access from across the robot program - this is important for facilitating convenience methods in both the `Command` and `Subsystem` interfaces, as well as not requiring users to inject a `Scheduler` across their entire codebase, which is tedious and provides little real benefit.  However, the new library does a much better job of avoiding rigid coupling to the `Scheduler` object, even though it is globally-acessible.

The `CommandScheduler` API is very simple:

##### Scheduling commands

```
void scheduleCommand(Command command, boolean interruptible);
void scheduleCommands(boolean interruptible, Command... commands);
```

The `interruptible` boolean sets if the command is permitted to be interrupted by a later-scheduled command that needs one of its required `Subsystems`.  When a `Command` is scheduled, the `CommandScheduler` first checks to see if its requirements are free.  If they are, the command is scheduled, its `initialize()` method is run, and its requirements are added to the list of currently-used requirements.  If its requirements are not all free, it is checked if all of the commands using requirements that it needs have been scheduled as `interruptible`; if they have, all of these commands are interrupted, their requirements are freed, and the command is scheduled as above.  If not, nothing is done.  The vararg version of the method simply runs the process repeatedly for multiple commands.

##### Running commands/subsystems

```
void run();
```

This functions almost exactly as before, except the actual responsibility of tracking the scheduling state of the commands now does belong to the `CommandScheduler`.

##### Canceling commands

```
void cancelCommand(Command... commands);
void cancelAll();
```

Forceably ends the given commands, if they are currently scheduled.  The commands will have their `interrupt()` methods called, rather than their `end()` methods.  Note that a command can be cancelled ever if it is not scheduled as `interruptible` - the `interruptible` tag *only* determines if the command can be interrupted by *another command* through its requirements.

##### Querying command state

```
boolean isScheduled(Command command);
double timeSinceScheduled(Command command);
Command requiring(Subsystem subsystem);
Command getDefaultCommand(Subsystem subsystem);
```

These methods are fairly self-explanatory, and provide ways for the user to get information about the state of various `Command`s or `Subsystem`s.  They are often wrapped through convenience methods in the `Command` and `Subsystem` interfaces.  Note that `isScheduled()` and `timeSinceScheduled()` do not provide information about Commands that have been composed within CommandGroups - the scheduler does not see CommandGroup internals.

##### Adding additional actions to perform on execution of command subroutines

````
void onCommandInitialize(Consumer<Command> action);
void onCommandExecute(Consumer<Command> action);
void onCommandInterrupt(Consumer<Command> action);
void onCommandEnd(Consumer<Command> action);
````

These provide a simple way to add a task for the scheduler to perform whenever a command is initialized, executed, interrupted, or ended.  For example, one could insert a logging call to mark the event.
