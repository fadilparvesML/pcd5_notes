
The Memento design pattern is a behavioral pattern that captures and externalizes an object's internal state so that the object can be restored to this state later, without violating encapsulation. This pattern is especially useful for implementing features such as undo mechanisms in applications.

### Example: Tracking and Restoring an F1 Car's Setup Changes

Imagine you're part of an F1 team's engineering crew. During practice sessions, you frequently adjust the car's setup to find the optimal configuration for the track. You need a way to record these setups and revert to any of them quickly.

#### Step 1: Memento

First, create a Memento class that will store the state of the Originator (the car's setup).

```python
class CarSetupMemento:
    def __init__(self, tire_pressure, suspension_height, wing_angle):
        self._tire_pressure = tire_pressure
        self._suspension_height = suspension_height
        self._wing_angle = wing_angle

    def get_saved_state(self):
        return (self._tire_pressure, self._suspension_height, self._wing_angle)
```

#### Step 2: Originator

The Originator class generates a memento object capturing its current state and can use such an object to restore its state.

```python
class CarSetup:
    def __init__(self):
        self.tire_pressure = None
        self.suspension_height = None
        self.wing_angle = None

    def set_setup(self, tire_pressure, suspension_height, wing_angle):
        self.tire_pressure = tire_pressure
        self.suspension_height = suspension_height
        self.wing_angle = wing_angle

    def save_to_memento(self):
        return CarSetupMemento(self.tire_pressure, self.suspension_height, self.wing_angle)

    def restore_from_memento(self, memento):
        self.tire_pressure, self.suspension_height, self.wing_angle = memento.get_saved_state()
```

#### Step 3: Caretaker

The Caretaker class keeps track of multiple mementos without examining their contents.

```python
class SetupHistory:
    def __init__(self):
        self._saved_setups = []

    def save_setup(self, memento):
        self._saved_setups.append(memento)

    def get_setup(self, index):
        return self._saved_setups[index]
```

#### Step 4: Client Code

Use these classes to save and restore car setups during a practice session.

```python
def main():
    car_setup = CarSetup()
    history = SetupHistory()

    # Making adjustments and saving setups
    car_setup.set_setup(tire_pressure=25, suspension_height=10, wing_angle=5)
    history.save_setup(car_setup.save_to_memento())

    car_setup.set_setup(22, 12, 3)  # New setup
    history.save_setup(car_setup.save_to_memento())

    # Restore to the first saved setup
    car_setup.restore_from_memento(history.get_setup(0))
    print(f"Restored setup: Tire Pressure={car_setup.tire_pressure}, Suspension Height={car_setup.suspension_height}, Wing Angle={car_setup.wing_angle}")

if __name__ == "__main__":
    main()
```

Output:

```
Restored setup: Tire Pressure=25, Suspension Height=10, Wing Angle=5
```

In this example, `CarSetup` is the Originator that can save and restore its state (setup configuration). `CarSetupMemento` is the Memento that holds the state of the Originator. `SetupHistory` acts as the Caretaker that maintains a history of states. The process allows the F1 team to revert the car's setup to any previously saved state, facilitating experimentation with different configurations during practice.