
[PyTest Official Documentation](https://docs.pytest.org/en/stable/)<br>
[PyTest API](https://docs.pytest.org/en/stable/reference/reference.html)<br>


## Running PyTest

Running pytest without mentioning a filename will run all files of format `test_*.py` or `*_test.py` in the current directory and subdirectories. Pytest automatically identifies those files as test files. We can make pytest run other filenames by explicitly mentioning them.

Pytest requires the test function names to start with `test`. Function names which are not of format test* are not considered as test functions by pytest. We cannot explicitly make pytest consider any function not starting with test as a test function.

### EXECUTION

#### to run all tests (in a directory)
```
pytest -v
```

#### to run a specific test (in a directory)
```
pytest <filename> -v
```

#### how to run only specific set of tests<br>
There are two ways PyTest allows this:
1. Select tests to run based on substring matching of test names<br>
    to execute tests containing a string in its name
    ```
    pytest -k <substring> -v
    ```
    where `-k <substring>` represents substring to search for in the test names
2. Select tests groups to run based on the markers applied.
    Pytest allows use of markers on test functions. Markers are used to set various features/attributes to test functions. Markers are applied on the tests using the syntax
    ```
    @pytest.mark.<markername>
    ```
    To run marked tests, use the following syntax:
    ```
    pytest -m <markername> -v
    ```

    ```python
    import pytest
    
    @pytest.mark.great
    def test_greater():
        num = 100
        assert num > 100

    @pytest.mark.great
    def test_greater_equal():
        num = 100
        assert num >= 100

    @pytest.mark.others
    def test_less():
        num = 100
        assert num < 200
    ```

### FIXTURES
Fixtures are fucntions, which will run before each test function to which it is applied. Fixtures are used to feed some data to the tests such as database connections, URLs to test and some sort of input data. Therefore, instead of running the same code for every test, we can attach fixture function to tests and it will run and return the data to the test before executing each tests.

A function is marked as fixture by:
```
@pytest.fixture
```

```python
import pytest

@pytest.fixture
def input_value():
   input = 39
   return input

def test_divisible_by_3(input_value):
   assert input_value % 3 == 0

def test_divisible_by_6(input_value):
   assert input_value % 6 == 0
```
Here we have a fixture function name `input_value`, which supplies the inpute to the test. To access the fixture function, the tests have to mention the fixure name as inpute parameter.

However, the approach comes with its own limitation. A fixture function defined inside a test file has a scope within the test file only. We cannot use that fixture in another test file. To make a fixture available to multiple test files, we have to define the fixture fucntion in a file called `conftest.py`.

Fixtures are great for extracting data or objects that you use across multiple tests. However, they aren’t always as good for tests that require slight variations in the data. Littering your test suite with fixtures is no better than littering it with plain data or objects. It might even be worse because of the added layer of indirection.

As with most abstractions, it takes some practice and thought to find the right level of fixture use.

Nevertheless, fixtures will likely be an integral part of your test suite. As your project grows in scope, the challenge of scale starts to come into the picture. One of the challenges facing any kind of tool is how it handles being used at scale, and luckily, pytest has a bunch of useful features that can help you manage the complexity that comes with growth.

### Conftest.py

We can define the fixture functions in thisz file to make them accessible across multiple test files. Create a new file `conftest.py`
```python
import pytest

@pytest.fixture
def input_value():
   input = 39
   return input
```
now test files can call `input_values`
```python
import pytest

def test_divisible_by_3(input_value):
   assert input_value % 3 == 0

def test_divisible_by_6(input_value):
   assert input_value % 6 == 0
```
The tests will look for fixture in the same file. As the fixture is not in the file, it will check for fixutre in `conftest.py`. `pytest` looks for a conftest.py module in each directory. If you add your general-purpose fixtures to the conftest.py module, then you’ll be able to use that fixture throughout the module’s parent directory and in any subdirectories without having to import it. This is a great place to put your most widely used fixtures.

### Parameterizing Tests

Parametrerizing of a test is done **to run the test against multiple sets of inputs**. We can do this by using the following marker:
```python
@pytest.mark.parameterize
```

```python
import pytest

@pytest.mark.parametrize("num, output",[(1,11),(2,22),(3,35),(4,44)])
def test_multiplication_11(num, output):
   assert 11*num == output
```
Here the test multiples an in put with 11 and compares the result with the expected output. The test has 4 sets of inputs, each has 2 values one os the number to be multiplied with 11 and the other is the expected result.

```python
@pytest.mark.parametrize("palindrome", [
    "",
    "a",
    "Bob",
    "Never odd or even",
    "Do geese see God?",
])
def test_is_palindrome(palindrome):
    assert is_palindrome(palindrome)

@pytest.mark.parametrize("non_palindrome", [
    "abc",
    "abab",
])
def test_is_palindrome_not_palindrome(non_palindrome):
    assert not is_palindrome(non_palindrome)
```

```python
@pytest.mark.parametrize("maybe_palindrome, expected_result", [
    ("", True),
    ("a", True),
    ("Bob", True),
    ("Never odd or even", True),
    ("Do geese see God?", True),
    ("abc", False),
    ("abab", False),
])
def test_is_palindrome(maybe_palindrome, expected_result):
    assert is_palindrome(maybe_palindrome) == expected_result
```

### Xfail & SKIP test

1. A test is not relevant for some time due to some reasons
2. A new feature is being implemented and we already added a test for that feature

In these situations, we have the option to xfail the test or skip the tests.
Pytest will execute the xfailed test, but it will not be considered as part failed or passed tests. Details of these tests will not be printed even if the test fails (pytest prints the failed test details). Use the following marker:
```python
@pytest.mark.xfail
```

Skipping a test means that the test will not be executed. We can skip tests using the following marker:
```python
@pytest.mark.skip
```

### Maxfail

The syntax to stop the test suite after it reaches a certain number of test fails is:

```python
pytest --maxfail = <num>
```

### Duratino Reports: Test Duration Record
To find out test execution time duration of tests by using `--durations`. `--durations` expects `n` value. `n` is the number of the slowest tests.


### Parallel Test Execution
By default pytest runs in sequential order. To overcome large test run time pytest offers option to run tests in parallel.

We firlst need to install `pytest-xdist` plugin
```python
pip install pytest-xdist
```

execute the test suite with 3 workers
```python
pytest -n 3
```


`-n <num>` runs the tests by using multiple workers, here 3.

### XML Output
We can generate details of the test execution in an xml file.
```python
pytest test_multiplication.py -v --junitxml="result.xml"
```

### Useful pytets plugins

`pytest-randomly` forces the tests to run in a random order. This is a great way to uncover tests that depends of running in a specific order, stateful dependency.

`pytest-cov`
`pytest-django`
`pytest-bdd`
