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

