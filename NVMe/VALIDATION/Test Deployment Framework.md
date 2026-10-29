
To create a Python script that deploys and executes various tests, I recommend using a *modular, object-oriented test framework.*

The main idea is to separate the logic that manages test execution from the individual tests themselves. This allows you to add, remove, select, and execute tests without modifying the main script.

For example, you could build a framework that executes NVMe validation tests such as FDP configuration, namespace management, RUH status, and PID validation.

## 1. Overall architecture

![[overall architecture.png]]

## 2. Recommended project structure

```
test_framework/
│
├── main.py
├── test_manager.py
├── base_test.py
├── config.py
│
├── tests/
│   ├── __init__.py
│   ├── test_fdp_enable.py
│   ├── test_ruh_status.py
│   ├── test_pid_validation.py
│   └── test_namespace.py
│
├── utils/
│   ├── nvme_commands.py
│   └── logger.py
│
└── reports/
```

Each component has a specific responsibility.

|Component|Responsibility|
|---|---|
|`main.py`|Entry point; accepts arguments|
|`test_manager.py`|Discovers, selects, and executes tests|
|`base_test.py`|Defines common test interface|
|`config.py`|Holds device and environment configuration|
|`tests/`|Contains individual test implementations|
|`utils/`|Shared NVMe commands, logging, helpers|
|`reports/`|Stores execution results|

## 3. Create a common base test class

Every test should inherit from a common base class.

This ensures every test follows the same execution structure.

`base_test.py`

```python 

from abc import ABC, abstractmethod

class BaseTest(ABC):
    def __init__(self, device):
        self.device = device

    def setup(self):
        """Prepare test environment."""
        pass

    @abstractmethod
    def run(self):
        """Execute test logic."""
        pass

    def cleanup(self):
        """Restore test environment."""
        pass

    def execute(self):
        try:
            self.setup()
            self.run()
            print(f"{self.__class__.__name__}: PASS")

        except Exception as e:
            print(f"{self.__class__.__name__}: FAIL - {e}")

        finally:
            self.cleanup()
```

The base class provides four important methods:

- `__init__()`: Receives shared parameters, such as the NVMe device.
- `setup()`: Prepares the test environment.
- `run()`: Contains the actual test logic.
- `cleanup()`: Releases resources and restores the environment.
    
The `execute()` method controls the lifecycle of every test. For a production framework, cleanup exceptions should also be captured so they cannot hide the original test failure.

## 4. Create individual test classes

Each test inherits from `BaseTest` and implements its own `run()` method.

`tests/test_fdp_enable.py`

```python
from base_test import BaseTest

class FDPEnableTest(BaseTest):
    def setup(self):
        print("Preparing FDP enable test")

    def run(self):
        print(f"Testing FDP enable on {self.device}")

        # Actual NVMe commands would go here.
        # Raise an exception if validation fails.

    def cleanup(self):
        print("Cleaning up FDP enable test")
```

`tests/test_ruh_status.py`

```python
from base_test import BaseTest

class RUHStatusTest(BaseTest):

    def run(self):
        print(f"Testing RUH status on {self.device}")

        # Retrieve RUH status.
        # Parse the returned data.
        # Compare against expected values.
```

These are structural examples, not completed NVMe validation tests. The `run()` methods must perform real assertions before a PASS result is meaningful.

## 5. Create a test manager

The Test Manager is responsible for registering and executing tests.

`test_manager.py`

```python
class TestManager:
    def __init__(self):
        self.tests = {}

    def register(self, name, test_class):
        self.tests[name] = test_class

    def list_tests(self):
        return list(self.tests.keys())

    def run_test(self, name, device):
        if name not in self.tests:
            raise ValueError(f"Unknown test: {name}")

        test = self.tests[name](device)
        test.execute()

    def run_all(self, device):
        for name in self.tests:
            print(f"\nRunning: {name}")
            self.run_test(name, device)
```

The dictionary `self.tests` acts as a test registry.

For example:

```json
{
    "fdp_enable": FDPEnableTest,
    "ruh_status": RUHStatusTest
}
```

The manager can dynamically instantiate a test class and execute it without needing to know its internal logic.

## 6. Create the main deployment script

`main.py`

```python
import argparse
from test_manager import TestManager
from tests.test_fdp_enable import FDPEnableTest
from tests.test_ruh_status import RUHStatusTest

def main():
    parser = argparse.ArgumentParser()

    parser.add_argument(
        "--device",
        required=True,
        help="NVMe device path"
    )

    parser.add_argument(
        "--test",
        default="all",
        help="Test name or 'all'"
    )

    args = parser.parse_args()

    manager = TestManager()

    manager.register("fdp_enable", FDPEnableTest)
    manager.register("ruh_status", RUHStatusTest)

    if args.test == "all":
        manager.run_all(args.device)
    else:
        manager.run_test(args.test, args.device)


if __name__ == "__main__":
    main()
```

### Example commands

Execute all tests:
```bash
python main.py --device /dev/nvme0 --test all
```

Execute only the FDP Enable test:
```bash
python main.py --device /dev/nvme0 --test fdp_enable
```

Execute only the RUH Status test:
```bash
python main.py --device /dev/nvme0 --test ruh_status
```

This design allows the same script to deploy different test suites based on user input.

## 7. Add dynamic test discovery

As the framework grows, manually importing and registering every test becomes inconvenient.

Instead, Python can discover test classes dynamically using `importlib` and `inspect`.

The workflow becomes:

![[workflow.png|393]]

With dynamic discovery, adding a new test may only require creating a new Python file in the `tests/` directory.

For example:

```
tests/
├── test_fdp_enable.py
├── test_ruh_status.py
├── test_pid_validation.py
├── test_namespace.py
└── test_fdp_events.py  # Newly added
```

The framework automatically discovers and registers the new test class, provided it follows the required interface.

## 8. Features to add for a production test deployment system

|Feature|Purpose|
|---|---|
|Dynamic discovery|Automatically locate test classes|
|Test filtering|Select by name, category, priority, or tags|
|Parameterization|Run the same test with multiple configurations|
|Dependencies|Run prerequisite tests before dependent tests|
|Logging|Capture commands, responses, and errors|
|Timeout handling|Prevent hung tests from blocking execution|
|Retry policy|Retry infrastructure failures selectively|
|Result reporting|Generate JSON, HTML, or JUnit XML|
|Parallel execution|Run independent tests concurrently|
|Resource locking|Prevent conflicting access to the same SSD|
|Test configuration|Read YAML/JSON configuration files|
|Environment cleanup|Restore the DUT after each test|
|Resume capability|Continue interrupted test campaigns|

For NVMe validation, destructive tests should also require explicit authorization and device confirmation. Tests that modify FDP configuration, namespaces, or firmware state should not run automatically on an arbitrary device.

## 9. Alternative: Use pytest as the execution engine

Instead of developing every part of the test manager yourself, you can use pytest. Pytest already provides test discovery, fixtures, parameterization, filtering, assertions, and reporting integrations.

For example:
```python
import pytest

@pytest.mark.fdp
def test_fdp_enable(nvme_device):
    # Perform real FDP enable validation
    pass


@pytest.mark.fdp
def test_ruh_status(nvme_device):
    # Perform real RUH status validation
    pass
```

Execute:
```
pytest tests/ -m fdp -v
```

For a large SSD validation system, a practical architecture is to use pytest for test execution while building your own NVMe command abstraction, device management, firmware diagnostics, and test configuration layers.

## 10. Choosing an architecture

|Approach|Advantages|Best suited for|
|---|---|---|
|Simple script|Fast to develop|Small fixed test sets|
|Custom class-based manager|Full execution control|Proprietary validation systems|
|Dynamic plugin framework|Extensible test deployment|Multiple teams contributing tests|
|pytest-based framework|Mature discovery and reporting|Automated regression and CI|
|Hybrid manager + pytest|Custom orchestration with standard testing|Large-scale SSD validation|

My recommendation: For an NVMe validation automation system, use a hybrid architecture: a central deployment manager that selects tests and prepares devices, with pytest executing individual test cases.

This separates test deployment (which tests run, on which device, with which firmware and configuration) from test execution (setup, NVMe commands, assertions, cleanup, and results).

That separation makes the framework scalable when different engineering teams need to contribute and deploy their own test content.