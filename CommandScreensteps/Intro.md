# Introduction to Command-Based Robot Programming

## What is "command-based" programming?

WPILib supports a robot programming methodology called "command-based" programming.  In general, "command-based" can refer both the general
programming paradigm, and to the set of WPILib library resources included to facilitate it.

"Commmand-based" programming is an example of what is known as a "design pattern."  It is a general way of organizing one's robot code
that is well-suited to a particular problem-space.  It is not the only way to write a robot program, but it is a very effective one; 
command-based robot code tend to be clean, extensible, and (with some tricks) easy to re-use from year to year.

The command-based paradigm is an example of what is known as "declarative programming."  Rather than directly specifying the entirety of
the iteration-by-iteration logic of the 
