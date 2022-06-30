# 2022-2023 Telemetry Rewrite Proposal

This is a design document for a proposed refactoring of WPILib telemetry features for the 2022-2023 season.

## Motivations

As WPILib has grown, its telemetry architecture has grown with it.  Heading into the 2022-2023 season, WPILib currently plans to support:

* A communications protocol (NetworkTables)
* Four dashboards (LabView dashboard, SmartDashboard, Shuffleboard, Glass)
* Two logging solutions (Shuffleboard recordings, on-RIO logging)
* A declarative object-oriented telemetry API (Sendable)
* A code-bypassing "debug mode" for sensors and actuators (LiveWindow)

While the WPILib telemetry ecosystem is reasonably functional (especially when combined with some amount of external tooling), it can be confusing from the outside due to the large amount of duplication in the above tools ("which of the four dashboards should I use and why?").  Additionally, as a result of having emerged from a shared set of "ancestral" features, many of the above tools have overlapping implementations that interact in surprising and sometimes undocumented ways.

Healthy software development generally follows two alternating phases: expansion and consolidation.  The WPI telemetry ecosystem has grown through steady expansion into a substantial and powerful set of features.  It is now in need of a consolidation phase to reduce the total API surface area.  Consolidating and refactoring the existing implementations will serve two purposes: it will provide new users with a clearer and more-intuitive "happy path" for their robot telemetry, and it will reduce the maintenance burden on on the WPILib development team by disentangling the implementations of unrelated API features and reducing the total volume of maintained code.

### Too many dashboards

The current API presents new users with a number of pain points.  Firstly, users are presented with a choice of four different dashboards - each with slightly-different featuresets, but without enough distinguishing features to clearly guide users as to which dashboard is useful in which situation.

In particular:

* The LabView dashboard and SmartDashboard are largely legacy features that have been totally supplanted in functionality by Shuffleboard and Glass.  Neither is likely to be easily maintainable in the event of a major breakage.  Both options are substantially clunkier and less-functional than the alternatives, and teams which choose these dashboards are probably worse-off for the choice.  Is continued support for legacy team-side dashboard code really worth this cost, especially given the tendency of FRC teams to rewrite most of their code anually?
* ShuffleBoard is clearly superior to Glass as an operating dashboard, and has many features (robot-code-driven tabs and layouts, data recording and playback) that make it appealing as the "default" robot dashboard.  However, it is already in danger of slipping into the "legacy code" state of the above two tools due to the migration of developer effort to Glass.  This could eventually cost teams if support for the dashboard becomes difficult to sustain while no other dashboard option offers a similar featureset.
* Glass is a *fantastic* tool for real-time plotting and signal analysis, but currently lacks a robust featureset to be usable as a dashboard.

Under the current ecosystem, effective users will likely want to make use of both Shuffleboard and Glass, while ignoring the LabView dashboard and SmartDashboard.  However, the documentation and API shapes do not necessarily lead users to this conclusion.  Moreover, developer effort is difficult to split between two tools, but the current distribution of features between tools demands it.

### Shuffleboard performance/maintenance status

Shuffleboard is, essentially, stable abandonware.  However, it has a number of known performance issues and complaints about stability, resource use, and general user experience remain frequent year-after-year.  Given the lack of JavaFX developer impetus/availability/expertise on the WPILib team, and the irrelevance of the platform to the current software "zeitgeist," this is a major stumbling block so long as Shuffleboard's core value-add features (user-code-configurable tab/layout telemetry/dashboard viewer) remain non-duplicated within the ecosystem.

### Overloading of the Sendable implementation

On the surface, the `Sendable` interface offers a powerful abstraction to teams who wish to log their state declaratively rather than imperatively.  However, behind-the-scenes `Sendable` pulls double-duty as the backbone of the `LiveWindow` implementation, which is responsible for auto-populating a list of sensors and actuators for easy debugging.

The `LiveWindow` design philosophy holds that all sensor and actuator objects in code should be accessible via the dashboard while the robot is in Test Mode, *regardless of what the user has done in their robot code.*  While this is a nice idea in theory, it places *huge* constraints on the handling of the `Sendable` interface which often interfere with the latter's use as a telemetry handler:

* While writes to the dashboard work as expected, reads from the dashboard (i.e. calls to the bound setters) are suspended *except* in LiveWindow mode.  This is not documented anywhere on the class, and is extremely inconvenient.
* The API includes an `isActuator` flag which seemingly serves no purpose other than to disable certain widgets on ShuffleBoard (it does not interact with the other dashboards).
* Sendables can be registered with `LiveWindow` even if they have no business being there (i.e. they are not a sensor or an actuator).
* It is not documented which `Sendable`s automatically add themselves to the `LiveWindow` registry immediately upon construction, and the application of the `Sendable` interface to certain classes which would otherwise be able to provide useful telemetry to teams (e.g. the various pneumatics classes) has been delayed or abandoned due to the complexities of maintaining a such a static registry, even when the classes could feasibly implement the interface and expose themselves to users sans automatic registration.
  
### Crude interaction between Sendable and Commands

Parts of the `Sendable` interface - particulary those involving the `LiveWindow` registry - have hard dependencies on the design philosophy of the Command-based framework.  In particular, `LiveWindow` widgets are grouped by "subsystem," even though they are populated automatically on robots that may not have recognizable `Subsystem` classes.  The relevant values are silently defaulted - and the functionality rendered useless - in all but a very narrow set of circumstances.

The command scheduler implements `NTSendable` in a fairly ad hoc way (the functionality was copied directly from the original command framework to allow backwards compatibility, which may have been a mistake).  This is one of the few major reliances on `NTSendable` remaining in the codebase, and may present an opportunity to remove the dependency entirely.

### Limited Sendable composability

While the NT architecture supports hierarchical structures (albeit indirectly through parsing of key names), the `Sendable` API only allows *flat* composition (excluding the aforementioned `Subsytem` tagging in the `LiveWindow` registry, which most teams are probably not aware of for the reasons mentioned in the previous section).  That is, child objects that implement `Sendable` can *only* be made to place their logged fields directly in the table entry of their parent.  This makes it very hard to scale `Sendable` in a sensible way, and can cause namespace collisions even in projects small enough to avoid the scaling issues.

### Poorly-documented LiveWindow functionality

LiveWindow achieves a coherent feature-set, but it is not well-documented what this feature-set actually is or why it contains the features that it does.  The intention that LiveWindow be used to bypass user code entirely is difficult to maintain from a code standpoint, and the cost is harder still to justify when most users do not seem to be aware of the design intent and tend to avoid the feature entirely.

### Poorly-documented Sendable functionality

`Sendable` only calls bound listeners when in `LiveWindow` mode, making the listener handles on `Sendablebuilder` mostly useless despite no indication of such in the API documentation or related tutorials.  As mentioned above, this is a wart due almost entirely due to the intermixture of `Sendable`'s telemetry and troubleshooting functionalities.

### Verbose Shuffleboard API

The Shuffleboard API is extremely powerful and allows more or less the entire dashboard layout to be declaratively structured from robot code.  The native API for this, however, is verbose and not self-documenting.  Simple declarations require long method chains, and widget arguments are passed via string key-value pairs without type safety.

### Lack of a universal metadata layer

Both the new datalogging implementation and NT4 include support for string-valued metadata.  The implementation of the code-driven layout configuration in Shuffleboard involves an ad-hoc formatting metadata layer that is not shared by any of the other tooling.

Since these metadata layers are incompatible, any effort spent on Shuffleboard widgets is useless for the newer Glass-based tooling.  Were these metadata layers compatible (ideally, identical), then Shuffleboard features can smoothly interop with Glass and other more-modern dashboards (including, potentially, a web-based successor to Shuffleboard).

This seems to have been the original intent of Shuffleboard's metadata structure and adherence to generic string-string maps for widget properties.  Building along these lines preserves much of the design work that went into the tool.

### Obscure calls for simple functions

The simplest way to log telemetry to the dashboard declaratively is through a `SmartDashboard` method, despite `SmartDashboard` being a nearly-obsolete legacy framework with no active maintenance.  As Glass does not have its own API, ahe `Shuffleboard` API is rather verbose, `SmartDashboard` calls are often used even in codebases that never actually open `SmartDashboard`.  This is not ideal.

Moreover, the various `put` functions for both `SmartDashboard` and `Shuffleboard` suddenly change semantics from immediate-mode periodic "put" calls to declarative databinding depending on whether the argument is a `Sendable`.  This overloading of the `put` name is highly confusing.

### Redundant APIs for defining subsystem/device properties

A subsystem (or device--there is very little difference at this level) currently needs to declare its properties multiple times: once for telemetry/NT, once for simulation, and possibly more. NT and simulation being relatively simialar isn't new, and is evident from classes such as `Field2d` and `Mechanism2d` being migrated rather easily from a simulation backend to an NT one. The WS setup used primarily for the Romi bots is also very similar to NT and sim.

While a longer-term goal, merging of NT and simulation would decrease boilerplate in both team and library code. Developing sim extensions/plugins would also become easier, partially due to HALSim extensions currently needing to be written in C++ while a connection to NT can be implemented in nearly any programming language (for starters, NT APIs exist for Java, Python, JS, and more).

To leverage NT for this purpose, we first need to clarify and refine the declarative methods for binding in-code properties to fields on NT.

### Values being sent multiple times under different keys

Assuming all data available is sent, there are often multiple values duplicated when a system is published. This is commonly due to re-sending of values with a different key which is shorter, more concise, and more expressive for the specific mechanism (this problem becomes even more complicated when a single system has multiple devices with similar properties--in that case, the last part of the key isn't enough to distinguish between readings). Another possible cause can be units: a situation where the data is originally published in some unit while the team wants the data in another. This causes redundant data to be sent over NT, a problem due to the bandwidth limit. A solution similar to symlinks (where a single, stable string value would point to the data instead of duplicating multiple fields) could handle the first cause. The second case, derived values, is more complicated but can still be achieved to some level using metadata.

## Solutions

The above problems are substantial.  Fortunately, many of the fixes are potentially simple, and build off of initiatives already being undertaken by current WPILib team members.

### Unified NT4/DataLogging/Shuffleboard metadata format

The new on-RIO datalog implementation ships with support for per-entry string-valued metadata.  The spec for NT4 includes support for the same. Both are ostensibly a JSON spec - this is convenient, because WPILib can "reserve" properties in the metadata JSON for its own purposes, and past that the spec can be left open for users to expand as they wish.  With the growing stability, accessibility, and popularity of web dashboards, a JSON spec will allow smooth interop with custom user solutions and potentially with future directions for WPILib itself.

This is a natural point of departure for a unified telemetry metadata spec - if Shuffleboard, in particular, can be migrated to this new spec, then the path forward for development and consolidation of dashboard tooling becomes vastly easier.  In particular, features from Shuffleboard that *must* be migrated include:

* Display name
* Size
* Position
* Tabs
* Layouts
* Widget type
* Data-specific display parameters

Features that are not yet supported, but which have obvious/immediate uses and would integrate cleanly with existing infrastructure include:

* Measured unit/dimension
* Update frequency (for telemetry)
* Command-based structure (subsystem tags)
* Actuation safety (for telemetry consumers)

Only some of these features are metadata of the telemetry topics themselves - others are metadata of dashboard display widget.  A tentative spec for metadata of a NT4 telemetry topic can be seen below:

```typescript
{
  // Display name (default, can be overridden at the widget level)
  name: string,
  // Unit of measure (if a scalar quantity)
  unit: string,
  // Update rate for robot-side code
  updatePeriodMs: number,
  // Name of the robot subsystem to which these data are associated
  subsystem: string,
  // Whether or not the value controls a robot actuation (if a scalar quantity)
  actuator: boolean
}
```

Dashboard widgets require slightly more involved handling, which is discussed below:

### Reserved metadata topic for dashboard layout definition

Shuffleboard defines tabs, layouts, and widgets in a set of metadata tables, which themselves are ordinary NT entries, and the tab/layout/widget hierarchy is represented by the pathstrings used for the entry keys.  These entries live under a reserved `metadata` content root.

With the addition of an explicit JSON metadata layer to NT4, we can replaec this entirely with a *single* JSON document that defines the structure of the dashboard, located under the `dashboard` topic.  This might look something like so:

```typescript
{
  tabs: [
    {
      name: 'drive',
      elements: [
        {
          // display name
          name: 'Heading',
          // widget type
          type: 'gyro',
          // we can point to the widget data by NT4 topic pathstring
          data: 'drive/heading'
        },
        {
          name: 'Wheel Velocities',
          type: 'graph',
          // multi-value plots easily represented in JSON with array type
          data: ['drive/leftVelocity', 'drive/rightVelocity'],
          // we can easily spec for display parameters per widget type
          graph: {
            limits: {
              x: [0, 10],
              y: [-50, 50]
            }
          }
        }
      ]
    }
  ]
}
```

A simple builder API, similar to the current Shuffleboard API, can allow dashboard configuration from robot code:

```java
var exampleTab = Telemetry.getDashboardTab("Drive");
exampleTab.addWidget(new Widgets.Gyro("Heading", "drive/heading"))
  // Graph widget ctor can be variadic to handle multiple data sources
  .addWidget(new Widgets.Graph("Wheel Speeds", "drive/leftVelocity", "drive/rightVelocity")
    .withXLimits(0, 10)
    .withYLimits(-50, 50));
```

The above differs from the Shuffleboard API in that it has direct programmatic representation for widgets in the form of the widget classes (e.g. `Widget.Gyro`).  Shuffleboard only exposes a generic string-valued `withProperties` decorator, which does not provide type safety for user-specified widget properties.

### Deprecate all existing SmartDashboard and Shuffleboard methods, replace with implementation-neutral equivalents

With an implementation-neutral metadata spec available, telemetry publishing is no longer tightly coupled to the user's choice of dashboard.  Accordingly, there is no need to maintain separate `SmartDashboard` and `Shuffleboard` APIs.  In fact, it is not necessary for the API shape itself to contain any reference to the dashboard implementation or spec - this will be done only where it is convenient and reduces boilerplate in user code.

The unified replacement API will look something like:

```java
// Simple imperative logging directly delegates to NT4
Telemetry.publishDouble("key", value);
// Can support dimensions through the API by delegating to metadata (C++ could use templates and be smarter)
Telemetry.publishQuantity("key", value, "meters");

// Declarative binding with suppliers/consumers
Telemetry.publishDouble("key", () -> value);
Telemetry.subscribeDouble("key", (double value) -> doThing(value));

// Declarative binding of mutable classes similar to `putData(Sendable)` in existing impl
// Optional final metadata string for aggregation
var telemetryBinding = Telemetry.bind("key", telemetryNode);
// Unbind later if needed (for cleanup etc)
telemetryBinding.unbind();
```

Metadata for a given topic will be deep-merged into the existing metadata as they are introduced.  There does not seem to be any need for a system to remove metadata - such an interaction would be a code smell.

### Replace Sendable with new, cleaner API

`Sendable` should be replaced by a new interface, `TelemetryNode`.  The API shape for this will be fundamentally similar to `Sendable`:

```java
public interface TelemetryNode {
  void bind(TelemetryBuilder builder);
}
```

The "bind" method fulfills the role of `initSendable`, with a few key differences:

Unlike `Sendable`, both `TelemetryNode` will support hierarchical nesting of implementing objects - that is, in addition to exposing binding handles for fields of primitive data type, it will allow *entire* `TelemetryNode`s to be added as children:

```java
@Override
public void bind(TelemetryBuilder builder) {
  // Add a `TelemetryNode` as a child; similarly-shaped API to ordinary binding
  builder.bind("child", childNode);
}
```

No auto-registration will be supported.  `TelemetryNode`s will only have their listeners bound if explicitly bound, either as a root node or as a child.

Publishing and subscribing will no longer be overloaded onto a single `addProperty` method of the builder, since they only rarely overlap in practice:

```java
@Override
public void bind(TelemetryBuilder builder) {
  // API here matches the ordinary Telemetry API
  builder
    .publishDouble("publishedValue", () -> value)
    .subscribeDouble("subscribedValue", (double value) -> { doThing(double); });
}
```

#### Breaking out of the object tree

It's sometimes useful to bind fields or children to paths that do not necessarily follow the in-code object tree - for example, to put certain data under a shared "diagnostics" aggregation even when it is held across different classes in code.

To do this, users can call `bind`, `publish`, or `subscribe` directly on `Telemetry` instead of on the `TelemetryBuilder`.  The APIs will be identically-shaped, and will differ only in the property path locations.

#### Annotation support (Java only)

There has been an intent to upstream [Oblog](http://github.com/oblarg/oblog), which is a Shuffleboard wrapper, for a number of years.  This was delayed due to COVID, and given the substantial shifts in the landscape since the last revision to Oblog (notably, NT4), the strategy for mainlining an annotation-based telemetry wrapper has become quite a bit simpler.

Oblog's core functionality revolves around two annotations: `@Log` and `@Config`.  The former is used to declaratively bind fields or getters to publish data to Shuffleboard (or NetworkTables), and the latter is used to bind fields or setters to listen to changes on a published field on Shuffleboard (or NetworkTables).

This design closely overlaps the existing `Sendable` implementation, and the migration to `TelemetryNode` makes the structure even more amenable to a direct adaptation.  Similarly to Oblog, two primary annotations are needed: `@Publish` and `@Subscribe`, which function largely as sugar for the methods seen above.

Annotation processing in Oblog is performed at runtime.  This is often avoided due to cost concerns, but Oblog caches all relevant reflection results on boot and there is essentially no additional performance cost subsequently.  This architecture should be used again, here - it is simple and it avoids many of the hard limitations (and tooling costs) of compile-time annotation processing.

Accordingly, annotation processing will be done upon initial binding of the `TelemetryNode` instance.  Upon a call to `Telemetry.bind`, WPILib will dynamically search for `@Publish` and `@Subscribe` annotations on class members, and perform the analogous function calls that the user would have otherwise defined in the body of `TelemetryNode.bind`. 

So, roughly,

```java
@Publish
double publishedValue;
```

will perform (through reflection)

```java
@Override
public void bind(TelemetryBuilder builder) {
  builder
    .publishDouble("publishedValue", () -> publishedValue);
}
```

Annotation-bound telemetry fields will be added *before* the call to the user-defined override of `TelemetryNode.bind`, so that the latter (and more expressive) syntax has the final say.

```java
@Publish
double graphedValue;
```

Metadata (such as dimensions) can be specified via annotation parameters:

```java
// equivalent to above
@Publish(
  unit="meters"
)
double graphedValue;
```

Since `TelemetryNode` natively supports hierarchical nesting, Oblog's previously-weighty reflection-based structure inference can be abandoned; it, too, can now be implemented as sugar on the existing `TelemetryNode` implementation through a final annotation, `@Bind`, with

```java
// Add a TelemetryNode as a child
@Bind
TelemetryNode child;
```

being roughly equivalent to

```java
@Override
public void bind(TelemetryBuilder builder) {
  builder
    .bind(child);
}
```

To break out of the object tree with annotations, pass a value to the optional "path" parameter of appropriate annotation:

```java
@Publish(path="diagnostics")
double publishedValue;
```

