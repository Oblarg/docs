# Introduction to Command-Based Robot Programming

## What is "command-based" programming?

WPILib supports a robot programming methodology called "command-based" programming.  In general, "command-based" can refer both the general programming paradigm, and to the set of WPILib library resources included to facilitate it.

"Command-based" programming is an example of what is known as a [design pattern.](https://en.wikipedia.org/wiki/Design_pattern)  It is a general way of organizing one's robot code that is well-suited to a particular problem-space.  It is not the only way to write a robot program, but it is a very effective one;  command-based robot code tend to be clean, extensible, and (with some tricks) easy to re-use from year to year.

The command-based paradigm is also an example of what is known as [declarative](https://en.wikipedia.org/wiki/Declarative_programming) programming.  In declarative programming, the emphasis is placed on *what* the program ought to do, rather than *how* the program ought to do it.  Thus, the command-based libraries allow users to define desired robot behaviors while minimizing the amount of iteration-by-iteration robot logic that they must write.  For example, in a command-based program, a user can specify that "the robot should perform an action when a button is pressed":

```java
aButton.whenPressed(intake::run);
```

In contrast, in an ordinary [imperative](https://en.wikipedia.org/wiki/Imperative_programming) program, the user would need to check the button state every iteration, and perform the appropriate action based on the state of the button.

```java
if(aButton.get()) {
  if(!pressed) {
    intake.run();
    pressed = true;
  } else {
    pressed = false;
  }
}
```

## Subsystems and commands

![image of subsystems and commands](https://media.screensteps.com/images/Wpilib/241892/1/rendered/B035F1D9-FDC6-43E4-9DFC-0E7E1919F3DB.png?AWSAccessKeyId=AKIAJRW37ULKKSXWY73Q&Expires=1554399753&Signature=QKRJuPulad7maQXXoYfGUwLZSzs%3D)

The command-based pattern is based around two core abstractions: **commands**, and **subsystems.**

**Subsystems** are the basic unit of robot organization in the design-based paradigm.  Subsystems [encapsulate](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)) lower-level robot hardware (such as motor controllers, sensors, and/or pneumatic actuators), and define the interfaces through which that hardware can be accessed by the rest of the robot code.  Subsystems allow users to "hide" the internal complexity of their actual hardware from the rest of their code - this both simplifies the rest of the robot code, and allows changes to the internal details of a subsystem without also changing the rest of the robot code.  Subsystems implement the `Subsystem` interface.

**Commands** define high-level robot actions or behaviors that utilize the methods defined by the subsystems.  A command is a simple [state machine](https://en.wikipedia.org/wiki/Finite-state_machine) that is either initializing, executing, ending, or idle.  Users write code specifying which action should be taken in each state.  Simple commands can be composed into "command groups" to accomplish more-complicated tasks.  Commands, including command groups, implement the `Command` interface.

## How commands are run

Commands are run by the `CommandScheduler`, a singleton class that is at the core of the command-based library.  The `CommandScheduler` is in charge of polling buttons for new commands to schedule, checking the resources required by those commands to avoid conflicts, executing currently-scheduled commands, and removing commands that have finished or been interrupted.  The scheduler's `run()` method may be called from any place in the user's code; it is generally recommended to call it from the `robotPeriodic()` method of the `Robot` class, which is run at a default frequency of 50Hz (once every 20ms).

Multiple commands can run concurrently, as long as they do not require the same resources on the robot.  Resource management is handled on a per-subsystem basis: commands may specify which subsystems they interact with, and the scheduler will never schedule more than one command requiring a given subsystem at a time.  this ensures that, for example, users will not end up with two different pieces of code attempting to set the same motor controller to different output values.  If a new command is scheduled that requires a subsystem that is already in use, it will either interrupt the currently-running command that requires that subsystem (if the command has been scheduled as interruptible), or else it will not be scheduled.

Subsystems also can be associated with "default commands" that will be automatically scheduled when no other command is currently using the subsystem.  This is useful for continuous "background" actions such as controlling the robot drive, or keeping an arm held at a setpoint.

TODO: replace this graphic with one that isn't wrong

![scheduler control flow diagram](https://media.screensteps.com/images/Wpilib/241892/1/rendered/10c3a71e-3789-4c88-b60d-4bb11c517109.png?AWSAccessKeyId=AKIAJRW37ULKKSXWY73Q&Expires=1554406576&Signature=sOHzcI9Pdeh48SQImzwE6ORrE%2Fc%3D)

When a command is scheduled, its `initialize()` method is called once.  Its `execute()` method is then called once per call to `CommandScheduler.getInstance().run()`.  A command is un-scheduled and has its `end(boolean interrupted)` method called when either its `isFinished()` method returns true, or else it is interrupted (either by another command with which it shares a required subsystem, or by being canceled).

## Command groups

It is often desirable to build complex commands from simple pieces.  This is achievable by [composing](https://en.wikipedia.org/wiki/Object_composition) commands into "command groups."  A command group is a command that contains multiple commands within it, which run either in parallel or in sequence.  The command-based library provides several types of command groups for teams to use, and users are encouraged to write their own, if desired.  As command groups themselves implement the `Command` interface, they are [recursively composeable](https://en.wikipedia.org/wiki/Object_composition#Recursive_composition) - one can include command groups *within* other command groups.  This provides an extremely powerful way of building complex robot actions with a simple library.

# Subsystems

Subsystems are the basic unit of robot organization in the command-based paradigm.  A subsystem is an abstraction for a collection of robot hardware that *operates together as a unit*.  Subsystems [encapsulate](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)) this hardware, "hiding" it from the rest of the robot code (e.g. commands) and restricting access to it except through the subsystem's public methods. Restricting the access in this way provides a single convenient place for code that might otherwise be duplicated in multiple places (such as scaling motor outputs or checking limit switches) if the subsystem internals were exposed.  It also allows changes to the specific details of how the subsystem works (the "implementation") to be isolated from the rest of robot code, making it far easier to make substantial changes if/when the design constraints change.

Subsystems also serve as the backbone of the `CommandScheduler`'s resource management system.  Commands may declare resource requirements by specifying which subsystems they interact with; the scheduler will never concurrently schedule more than one command that requires a given subsystem.  An attempt to schedule a command that requires a subsystem that is already-in-use will either interrupt the currently-running command (if the command has been scheduled as interruptible), or else be ignored.

Subsystems can be associated with "default commands" that will be automatically scheduled when no other command is currently using the subsystem.  This is useful for continuous "background" actions such as controlling the robot drive, or keeping an arm held at a setpoint.  Similar functionality can be achieved in the subsystem's `periodic()` method, which is run once per run of the scheduler; teams should try to be consistent within their codebase about which functionality is achieved through either of these methods.

## Creating a subsystem

All that is needed to create a subsystem is to create a class that implements the `Subsystem` interface:

```java
import edu.wpi.first.wpilibj.experimental.command.Subsystem;

public class ExampleSubsystem implements Subsystem {
  // Your subsystem code goes here!
  
  public ExampleSubsystem() {
    register(); // Registers this subsystem with the scheduler so that its periodic method will be called.
  }
}
```

Notice that all methods in `Subsystem` are defaulted, so the user is not *required* to override anything.  While this is an entirely reasonable and correct way to create a subsystem, it has a few minor drawbacks: in particular, subsystems must call their `register()` method to register themselves with the scheduler in order for their periodic methods to be called when the scheduler runs (otherwise, the scheduler has no way of knowing that they are there!).  Additionally, users may want to leverage the ability to log their subsystem's current status on the robot dashboard, which is not supported by the baseline `Subsystem` interface.  To address this, new users are encouraged to instead subclass the abstract `SendableSubsystemBase` class, which implements the `Subsystem` interface and automatically registers the subsystem with the scheduler upon construction, as well as implementing the `Sendable` interface so that it can be sent to the robot dashboard:

```java
import edu.wpi.first.wpilibj.experimental.command.SendableSubsystemBase;

public class ExampleSubsystem extends SendableSubsystemBase {
  // Your subsystem code goes here!
}
```

## Simple subsystem example

What might a functional subsystem look like in practice?  Below is a simple pneumatically-actuated hatch mechanism from the HatchBot example project (TODO: link to it):

```java
package edu.wpi.first.wpilibj.examples.hatchbottraditional.subsystems;

import edu.wpi.first.wpilibj.DoubleSolenoid;
import edu.wpi.first.wpilibj.experimental.command.SendableSubsystemBase;

import static edu.wpi.first.wpilibj.DoubleSolenoid.Value.*;
import static edu.wpi.first.wpilibj.examples.hatchbottraditional.Constants.HatchConstants.*;

/**
 * A hatch mechanism actuated by a single {@link DoubleSolenoid}.
 */
public class HatchSubsystem extends SendableSubsystemBase {

  private final DoubleSolenoid m_hatchSolenoid =
      new DoubleSolenoid(kHatchSolenoidModule, kHatchSolenoidPorts[0], kHatchSolenoidPorts[1]);

  /**
   * Grabs the hatch.
   */
  public void grabHatch() {
    m_hatchSolenoid.set(kForward);
  }

  /**
   * Releases the hatch.
   */
  public void releaseHatch() {
    m_hatchSolenoid.set(kReverse);
  }
}
```

Notice that the subsystem hides the presence of the DoubleSolenoid from outside code (it is declared `private`), and instead publicly exposes two higher-level, descriptive robot actions: `grabHatch()` and `releaseHatch()`.  It is extremely important that "implementation details" such as the double solenoid be "hidden" in this manner; this ensures that code outside the subsystem will never cause the solenoid to be in an unexpected state.  It also allows the user to change the implementation (for instance, a motor could be used instead of a pneumatic) without any of the code outside of the subsystem having to change with it.

## Setting default commands

Setting a default command for a subsystem is very easy; one simply calls `Scheduler.getInstance().setDefaultCommand()`, or, more simply,  the `setDefaultCommand()` method of the `Subsystem` interface:

```java
Scheduler.getInstance().setDefaultCommand(driveSubsystem, defaultDriveCommand);
```

```java
driveSubsystem.setDefaultCommand(defaultDriveCommand);
```

# Commands

Commands are simple state machines that perform high-level robot functions with the methods defined by subsystems.  Commands can be either idle, in which they do nothing, or scheduled, in which the scheduler will execute a specific set of the command's code depending on the state of the command.  The `CommandScheduler` recognizes scheduled commands as being in one of three states: initializing, executing, or ending.  Commands specify what is done in each of these states through the `initialize()`, `execute()` and `end()` methods.

## Creating commands

As with subsystems, to create a command, a user only needs to make a class implementing the `Command` interface:

```java
import java.util.Collections;

import edu.wpi.first.wpilibj.experimental.command.Command;

public class ExampleCommand implements Command {
  // Your command code goes here!
  
  // Must be overridden!
  @override
  public List<Subsystem> getRequirements() {
    // What to do if you have no subsystem to require
    return Collections.emptySet();
  }
}
```

Again, as with subsystems, this is a perfectly fine way to create a command.  However, it is also slightly inconvenient.  Users are forced to override the `getRequirements()` method to declare subsystem requirements, which is cumbersome, and also as with subsystems users may wish to leverage the ability to send their commands to the dashboard (this provides a handy way to schedule commands for testing, as they can then be started from the dashboard).  It is therefore recommended that new users instead create commands by subclassing the abstract `SendableCommandBase` class:

```java
import edu.wpi.first.wpilibj.experimental.command.SendableCommandBase;

public class ExampleCommand extends SendableCommandBase {
  // Your command code goes here!
}
```

Requirements can then be declared simply by calling the `addRequirements()` method.

## The structure of a command

While subsystems are fairly freeform, and may generally look like whatever the user wishes them to, commands are quite a bit more constrained.  Command code must specify what the command will do in each of its possible states.  This is done by overriding the `initialize()`, `execute()`, and `end()` methods.  Additionally, a command must be able to tell the scheduler when (if ever) it has finished execution - this is done by overriding the `isFinished()` method.  All of these methods are defaulted to reduce clutter in user code: `initialize()`, `execute()`, and `end()` are defaulted to simply do nothing, while `isFinishsed()` is defaulted to return false (resulting in a command that never ends).

### Initialization

```java
@override
public void initialize() {
  // Code here will be executed when a command initializes!
}
```

The `initialize()` method is run exactly once per time a command is scheduled, as part of the scheduler's `schedule()` method.  The scheduler's `run()` method does not need to be called for the `initialize()` method to run.  The initialize block should be used to place the command in a known starting state for execution.  It is also useful for performing tasks that only need to be performed once per time scheduled, such as setting motors to run at a constant speed or setting the state of a solenoid actuator.

### Execution

```java
@override
public void execute() {
  // Code here will be executed every time the scheduler runs while the command is scheduled!
}
```

The `execute()` method is called repeatedly while the command is scheduled, whenever the scheduler's `run()` method is called (this is generally done in the main robot periodic method, which runs every 20ms by default).  The execute block should be used for any task that needs to be done continually while the command is scheduled, such as updating motor outputs to match joystick inputs, or using the ouput of a control loop.

### Ending

```java
@override
public void end(boolean interrupted) {
  // Code here will be executed whenever the command ends, whether it finishes normally or is interrupted!
  if (interrupted) {
    // Using the argument of the method allows users to do different actions depending on whether the command 
    // finished normally or was interrupted!
  }
}
```

The `end()` method of the command is called once when the command ends, whether it finishes normally (i.e. `isFinished()` returned true) or it was interrupted (either by another command or by being explicitly canceled).  The method argument specifies the manner in which the command ended; users can use this to differentiate the behavior of their command end accordingly.  The end block should be used to "wrap up" command state in a neat way, such as setting motors back to zero or reverting a solenoid actuator to a "default" state.

### Specifying end conditions

```java
@override
public boolean isFinished() {
  // This return value will specify whether the command has finished!  The default is "false," which will make the
  // command never end.
  return false;
}
```

Just like `execute()`, the `isFinished()` method of the command is called repeatedly, whenever the scheduler's `run()` method is called.  As soon as it returns true, the command's `end()` method is called and it is un-scheduled.  The `isFinished()` method is called *after* the `execute()` method, so the command *will* execute once on the same iteration that it is un-scheduled.

## Simple command example

What might a functional command look like in practice?  As before, below is a simple command from the HatchBot example project that uses the `HatchSubsystem` introduced in the previous section:

```java
package edu.wpi.first.wpilibj.examples.hatchbottraditional.commands;

import edu.wpi.first.wpilibj.examples.hatchbottraditional.subsystems.HatchSubsystem;
import edu.wpi.first.wpilibj.experimental.command.SendableCommandBase;

/**
 * A simple command that grabs a hatch with the {@link HatchSubsystem}.  Written explicitly for 
 * pedagogical purposes; actual code should inline a command this simple with 
 * {@link edu.wpi.first.wpilibj.experimental.command.InstantCommand}.
 */
public class GrabHatch extends SendableCommandBase {
  
  // The subsystem the command runs on
  private final HatchSubsystem m_hatchSubsystem;
  
  public GrabHatch(HatchSubsystem subsystem) {
    m_hatchSubsystem = subsystem;
    addRequirements(m_hatchSubsystem);
  }

  @Override
  public void initialize() {
    m_hatchSubsystem.grabHatch();
  }

  @Override
  public boolean isFinished() {
    return true;
  }
}
```

Notice that the hatch subsystem used by the command is passed into the command through the command's constructor.  This is a pattern called [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection), and allows users to avoid declaring their subsystems as global variables.  This is widely accepted as a best-practice - the reasoning behind this will be discussed in a later section (TODO: link to the section once it's written).

Notice also that the above command calls the subsystem method once from initialize, and then immediately ends (as `isFinished()` simply returns true).  This is typical for commands that toggle the states of subsystems, and in fact the command-based library includes code to make commands like this even more succinctly (TODO: link to section on this).

What about a more complicated case?  Below is a drive command, from the same example project:

```java
package edu.wpi.first.wpilibj.examples.hatchbottraditional.commands;

import java.util.function.DoubleSupplier;

import edu.wpi.first.wpilibj.examples.hatchbottraditional.subsystems.DriveSubsystem;
import edu.wpi.first.wpilibj.experimental.command.SendableCommandBase;

/**
 * A command to drive the robot with joystick input (passed in as {@link DoubleSupplier}s).
 * Written explicitly for pedagogical purposes - actual code should inline a command this simple
 * with {@link edu.wpi.first.wpilibj.experimental.command.RunCommand}.
 */
public class DefaultDrive extends SendableCommandBase {

  private final DriveSubsystem m_drive;
  private final DoubleSupplier m_forward;
  private final DoubleSupplier m_rotation;

  public DefaultDrive(DriveSubsystem subsystem, DoubleSupplier forward, DoubleSupplier rotation){
    m_drive = subsystem;
    m_forward = forward;
    m_rotation = rotation;
    addRequirements(m_drive);
  }

  @Override
  public void execute() {
    m_drive.arcadeDrive(m_forward.getAsDouble(), m_rotation.getAsDouble());
  }
}
```

Notice that this command does not override `isFinished()`, and thus will never end; this is the norm for commands that are intended to be used as default commands (and, as can be guessed, the library includes tools to make this kind of command easier to write, too!) (TODO: add link to relevant section).
