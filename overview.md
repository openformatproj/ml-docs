- [openformatproj/ml Simulation Framework](#openformatprojml-simulation-framework)
  - [Core Concepts](#core-concepts)
    - [`Part`](#part)
    - [`Port`](#port)
    - [`EventQueue`](#eventqueue)
    - [`Interface` \& `EventInterface`](#interface--eventinterface)
    - [`EventSource`](#eventsource)
    - [`Tracer`](#tracer)
  - [Execution Model](#execution-model)
    - [Example](#example)
    - [Other Event Transfers](#other-event-transfers)
    - [Idle State and Waking Up](#idle-state-and-waking-up)
  - [Standard Library Parts](#standard-library-parts)
    - [`EventToDataSynchronizer`](#eventtodatasynchronizer)
    - [`Operator`](#operator)
    - [`PID`](#pid)
    - [`Control_Element`](#control_element)
  - [Key Features](#key-features)
  - [Example: A Simple Simulation](#example-a-simple-simulation)
  - [Generating API Documentation](#generating-api-documentation)

# openformatproj/ml Simulation Framework

A lightweight, multi-threaded Python framework for building and running discrete-event and dataflow simulations. It is designed for creating complex, hierarchical systems where components communicate through well-defined interfaces.

## Core Concepts

The framework is built around a few key abstractions:

### `Part`
The fundamental building block of any simulation, and the owner of `Port`s and `EventQueue`s. A `Part` can be one of two types:

-   **Structural Part**: A container for other `Part`s. It defines the system's architecture by composing and connecting smaller components. It manages an execution loop that schedules its inner parts.
-   **Behavioral Part**: A component with a user-defined `behavior()` method. This is where the actual logic of your simulation (e.g., processing data, making decisions) resides.

### `Port`
A `Port` is a synchronous, one-to-many data interface. It enforces a strict "write-once, read-once" data transfer policy. Data written to a master port is broadcast to all connected slave ports.

The connection establishes a **master/slave** relationship where the **master is the source of the data** and the **slave is the destination**. This relationship depends on the connection pattern:

-   **Peer-to-Peer**: An `OUT` port (master) connects to an `IN` port (slave).
-   **Downward (Parent-to-Child)**: A parent's `IN` port (master) connects to a child's `IN` port (slave).
-   **Upward (Child-to-Parent)**: A child's `OUT` port (master) connects to a parent's `OUT` port (slave).

A `Port` is primarily manipulated via two methods:
-   `set(payload)`: Used within a `behavior()` method to place a value on an `OUT` port.
-   `get()`: Used within a `behavior()` method to read a value from an `IN` port. The data is considered consumed for the current execution cycle; the framework automatically clears the port's `updated` flag after the `behavior()` method has finished, preventing the part from re-executing on the same stale data.

To ensure robust data flow, `Port`s will raise exceptions on improper use:
-   `Port.OverwriteError`: Raised if `set()` is called on a port that already has unconsumed data.
-   `Port.StaleReadError`: Raised if `get()` is called on a port whose data has already been consumed (or was never set).

### `EventQueue`
An `EventQueue` is an asynchronous, buffered channel for passing event payloads between `Part`s.

It can be configured to operate in one of two modes:
-   **FIFO (default)**: The queue acts like a traditional queue. This mode is essential when every event must be processed in the order it was received, such as when simulating a system that must not drop any data from a real-world sensor.
-   **LIFO**: The queue acts like a stack. This is ideal for state-based simulations where only the most recent data is relevant (e.g., the latest sensor reading), as it naturally discards stale data during backpressure.
Like `Port`s, `EventQueue`s use a master/slave relationship for connections:
- **Source-to-Part**: An `EventSource`'s `OUT` queue (master) connects to a `Part`'s `IN` queue (slave).
- **Parent-to-Child**: A parent `Part`'s `IN` queue (master) can be routed to a child `Part`'s `IN` queue (slave).
### `Interface` & `EventInterface`
These are the internal "wires" of the framework, created automatically when you call `connect()` or `connect_event_queue()`. An `Interface` connects two `Port`s, and an `EventInterface` connects two `EventQueue`s. They are responsible for the actual data transfer logic.

### `EventSource`
An `EventSource` is a component that generates events from outside the main simulation loop. When its `start()` method is called, it launches its logic in a **new, dedicated thread**. The built-in `Timer` is a perfect example, emitting time events at a regular interval. Event sources are the primary way to drive a simulation.

Event transfers from a source to a part happen *immediately* in the `EventSource`'s thread when `emit()` is called. The behavior when the downstream `EventQueue` is full is controlled by the `on_full` parameter passed to the `EventSource`'s constructor (`FAIL`, `DROP`, or `OVERWRITE`), preventing the source from getting stuck while giving the user full control over the desired real-time behavior.
### `Tracer`
A centralized, thread-safe logging utility. It captures events from all components, sorts them by timestamp, and prints them in a clean, column-aligned format using fully-qualified component identifiers. This structured logging is invaluable for debugging the complex interactions in a multi-threaded simulation.

---

## Execution Model

A top-level **Structural Part** is designed to run a simulation loop. Calling its `start()` method launches its internal run loop in a **new, dedicated thread**. Each iteration of this loop performs a `_step()`, which consists of three main phases:

1.  **Scheduling Phase**: The `_step()` method scans its direct inner parts. A part is scheduled for execution if:
    -   Its `scheduling_condition` (if provided) returns `True`.
    -   If no condition is provided, it defaults to being schedulable if any of its `IN` ports has new data or any of its `IN` event queues is not empty.

2.  **Execution Phase**: If any parts were scheduled, the `execution_strategy` (e.g., `sequential_execution`) is called. It runs the `execute()` method on each scheduled part.
    -   **Behavioral Part `execute()`**: Calls the user-defined `behavior()` method. After the `behavior` method completes, the framework automatically clears the `updated` flag on any input ports that were read, preventing re-execution on the same data.
    -   **Structural Part `execute()`**: This enables hierarchical execution. It first transfers events and data from its own input interfaces to its children's, then recursively calls its *own* `_step()` method to run a full simulation cycle for its sub-system.

3.  **Dataflow Propagation Phase**: After the `Execution Phase` is complete, the `_step()` method calls `_transfer_and_clear_ports()`. This is where the synchronous dataflow happens. It copies the payload from all updated `OUT` ports to their connected `IN` ports. This sets the `updated` flag on the `IN` ports, making them ready for the *next* step's `Scheduling Phase`.

### Example
The framework's execution model is hierarchical and event-driven, designed for efficiency. The best way to understand it is through an example.

Imagine a simulation with a top-level structural part named `Top`. `Top` contains two children: a `Generator` and a `Processor`. The `Processor` is also a structural part, and it contains its own children, `Child_A` and `Child_B`. The data flows from the `Generator` to the `Processor`, which then routes it internally to `Child_A`, whose output then feeds `Child_B`.

The execution proceeds as follows:

1.  **Activation**: The `Generator` runs (perhaps driven by an external `EventSource`), producing data and placing it on the `Processor`'s input port.

2.  **Parent's Turn (`Top`)**: In its main execution loop, `Top` checks which of its direct children are ready to run. It sees that `Processor` has new input data (its input port is updated), so it schedules `Processor` for execution. `Top` has no knowledge of `Child_A` or `Child_B`; it only manages its direct children.

3.  **Child's Turn (`Processor`)**: `Top` calls `execute()` on `Processor`. Now, `Processor` takes control and begins its own internal execution loop.

4.  **Internal Propagation**: The `Processor` first transfers the data from its own input port to `Child_A`'s input port. Then, its internal loop runs:
    *   It sees `Child_A` is ready (its input port is updated) and executes it. `Child_A` processes the data and produces a new output for `Child_B`.
    *   The `Processor`'s loop continues. It now sees `Child_B` is ready (its input port is updated) and executes it.
    *   The loop runs again. This time, no more children have new data (their input ports are not updated). The internal system is idle, so the `Processor`'s internal loop finishes.

5.  **Return Control**: The `Processor`'s `execute()` method completes, and control returns to `Top`.

This hierarchical delegation is the key to the framework's efficiency. `Top` only needed to activate `Processor` once. The entire internal chain reaction was handled by `Processor` itself. If a parent part is not scheduled, its `execute()` method is never called, and its entire sub-system remains dormant, saving computational resources.

When the top-level part finds no children to run, the system is considered **idle**. The main simulation thread will then pause, waiting for an external `EventSource` (like a `Timer`) to provide a new event and wake it up.



### Other Event Transfers

-   **`EventSource` to `Part` (Asynchronous)**: This happens immediately in the `EventSource`'s thread whenever `emit()` is called. This is how external events enter the simulation.
-   **On Wakeup**: Immediately after the `run()` loop wakes up from an idle state, it transfers events from the top-level part's `IN` queues to its children's queues before starting a new `_step()`.

### Idle State and Waking Up

If the **Scheduling Phase** finds no parts to run at the top level, the system is considered **idle**. The run loop will then call `__wait_for_events()`, which pauses the simulation thread until an `EventSource` provides an event. When it wakes up, it first performs the `Part` to `Part` event transfer before beginning a new `_step()`.

This top-down, hierarchical model is highly efficient. If a parent part is not scheduled, its `execute()` method is never called, and therefore its entire sub-system (all of its inner parts) remains dormant.

This model is highly efficient, as it only executes components that have work to do and sleeps when the system is idle.

---

## Standard Library Parts

The framework includes a library of pre-built, reusable parts in `parts.py` to accelerate development.

### `EventToDataSynchronizer`
A generic behavioral part that synchronizes an asynchronous event to the synchronous dataflow domain. It consumes an event from an input `EventQueue`, extracts its payload, and sets it on an output `Port`.

### `Operator`
A generic, stateless behavioral part that applies a user-defined function to its inputs. It waits for all of its input ports to receive new data, then calls the provided function.

### `PID`
A behavioral part implementing a standard PID (Proportional-Integral-Derivative) controller. It dynamically calculates the time step (`dt`) from a `time` input port, making it robust and easy to use.
Its gains (Kp, Ki, Kd) can also be updated dynamically via input ports, making it suitable for adaptive control systems.

### `Control_Element`
A structural part that composes a `PID` controller and an `Operator` to form a complete, standard control loop element. It takes `setpoint` and `measure` inputs and produces an `actuation` output, ready to be connected to a simulated plant.

---

## Key Features

-   **Hierarchical Composition & Execution**: Build complex systems by nesting `Part`s. The execution model is also hierarchical, ensuring that entire branches of the system are skipped if their parent part is not active, leading to high efficiency.
-   **Event-Driven & Dataflow Architecture**: Combines asynchronous event handling with synchronous data processing.
-   **Multi-Threaded by Design**: Clear separation between event-generating threads and the main simulation thread.
-   **Robust Exception Handling**: Exceptions in any `EventSource` or `Part` thread are caught, logged, and can be inspected, preventing the entire application from crashing.
-   **Powerful Structured Logging**: The `Tracer` provides deep insight into the simulation's behavior, making debugging easy.
-   **Configurable Log Levels**: Easily control the verbosity of framework and user logs from a central `conf.py` file.

---

## Example: A Simple Simulation

The `demo.py` file provides a clear example. Here's a simplified look at its structure.

**1. Define a Behavioral Part**

This part has one input port and logs the time value it receives.

```python
class Consumer(Part):
    """A simple behavioral part that consumes and prints a time value."""
    def __init__(self, identifier: str):
        ports = [Port('time_in', Port.IN)]
        super().__init__(identifier=identifier, ports=ports)

    def behavior(self):
        # The scheduler guarantees the port is updated before calling behavior.
        time_port = self.get_port('time_in')
        time_value = time_port.get() # get() consumes the input
        self.trace_log("Time value received", details={"time": f"{time_value:.3f}"})
```

**2. Define a Structural Part to Compose the System**

This part contains other parts and wires them together.

```python
class Simulation(Part):
    """
    A structural part representing the entire simulation.
    """
    def __init__(self, identifier: str):
        # Define the inner components
        parts = {
            EventToDataSynchronizer(
                'time_dist',
                input_queue_id='time_event_in',
                output_port_id='time_out'
            ),
            Consumer('part_a'),
            Consumer('part_b')
        }
        
        super().__init__(
            identifier=identifier,
            parts=parts,
            event_queues=[EventQueue('main_time_q', EventQueue.IN, size=1)],
            execution_strategy=sequential_execution
        )
        
        # Get references to the parts
        distributor = self.get_part('time_dist')
        part_a = self.get_part('part_a')
        part_b = self.get_part('part_b')
        
        # Wire everything together. The queue and port IDs used here must
        # match those used to initialize the EventToDataSynchronizer.
        self.connect_event_queue(
            self.get_event_queue('main_time_q'),
            distributor.get_event_queue('time_event_in')
        )
        
        distributor_out_port = distributor.get_port('time_out')
        self.connect(distributor_out_port, part_a.get_port('time_in'))
        self.connect(distributor_out_port, part_b.get_port('time_in'))
```

**3. Run the Simulation**

The main block initializes the `Tracer`, the `Simulation`, and the `Timer`, connects them, and starts the threads.

```python
if __name__ == "__main__":
    Tracer.start(...)
    sim = Simulation('my_sim')
    timer = Timer(identifier='main_timer', interval_seconds=0.001, duration_seconds=0.1)

    sim.connect_event_source(timer, 'main_time_q')

    sim.start(stop_condition=lambda p: timer.stop_event.is_set())
    timer.start()

    sim.join()
    timer.join()
    Tracer.stop()
```

## Generating API Documentation

The project is set up with Sphinx to generate professional API documentation from the code's docstrings.

1.  Install Sphinx: `pip install sphinx sphinx_rtd_theme`
2.  Navigate to the `docs/` directory.
3.  Run `make html`.
4.  Open `docs/build/html/index.html` in your browser.