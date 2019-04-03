# Introduction to Command-Based Robot Programming

## What is "command-based" programming?

WPILib supports a robot programming methodology called "command-based" programming.  In general, "command-based" can refer both the general programming paradigm, and to the set of WPILib library resources included to facilitate it.

"Command-based" programming is an example of what is known as a "design pattern."  It is a general way of organizing one's robot code that is well-suited to a particular problem-space.  It is not the only way to write a robot program, but it is a very effective one;  command-based robot code tend to be clean, extensible, and (with some tricks) easy to re-use from year to year.

The command-based paradigm is an example of what is known as *declarative* programming.  In declarative programming, the emphasis is placed on *what* the program ought to do, rather than *how* the program ought to do it.  Thus, the command-based libraries allow users to define desired robot behaviors while minimizing the amount of iteration-by-iteration robot logic that they must write.  For example, in a command-based program, a user can specify that "the robot should perform an action when a button is pressed":

```java
aButton.whenPressed(intake::run);
```

In contrast, in an ordinary *imperative* program, the user would need to check the button state every iteration, and perform the appropriate action based on the state of the button.

```java
if(button.get()) {
  if(!pressed) {
    intake.run();
    pressed = true;
  } else {
    pressed = false;
}
```
