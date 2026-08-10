The **`traceback` module in Python** is ==a built-in library used to extract, format, and print stack traces of programs==. It allows you to programmatically inspect the chain of function calls that led to an error, which is highly useful for custom error logging and debugging. [1](https://docs.python.org/3/library/traceback.html), [2](https://docs.python.org/3/library/traceback.html), [3](https://python.plainenglish.io/deciphering-pythons-traceback-most-recent-call-last-aeea847bb34b), [4](https://oit.tudelft.nl/learn-python/2025/errors/traceback.html), [5](https://purpletutor.com/python-attributeerror/)

Key Capabilities

- **Print standard traces**: Replicates the exact error output Python shows when a script crashes.
- **Extract raw details**: Fetches filenames, line numbers, and function names as structured data.
- **Format to strings**: Converts stack traces into strings instead of printing them directly to the console. [1](https://medium.com/swlh/python-errors-done-right-faa1bfa85d02), [2](https://realpython.com/debug-python-errors/), [3](https://pymotw.com/3/traceback/index.html), [4](https://ipython.readthedocs.io/en/9.11.0/api/generated/IPython.core.ultratb.html)


```python
import traceback
import sys

def cause_error():
    return 1 / 0

def run_process():
    cause_error()

try:
    run_process()
except ZeroDivisionError:
    print("--- 1. Print directly to stderr ---")
    traceback.print_exc()

    print("\n--- 2. Get trace as a string (Good for logging) ---")
    error_string = traceback.format_exc()
    print(error_string)
    print("\n--- 3. Extract structured frames ---")
    # Gets the current execution frames
    frames = traceback.extract_tb(sys.exc_info()[2])
    for frame in frames:
        print(f"File: {frame.filename}, Line: {frame.lineno}, In: {frame.name}")
```


result
```powershell
PS C:\Users\danny.hur\Desktop> python3 .\traceback-test.py
--- 1. Print directly to stderr ---
Traceback (most recent call last):
  File "C:\Users\danny.hur\Desktop\traceback-test.py", line 11, in <module>
    run_process()
    ~~~~~~~~~~~^^
  File "C:\Users\danny.hur\Desktop\traceback-test.py", line 8, in run_process
    cause_error()
    ~~~~~~~~~~~^^
  File "C:\Users\danny.hur\Desktop\traceback-test.py", line 5, in cause_error
    return 1 / 0
           ~~^~~
ZeroDivisionError: division by zero

--- 2. Get trace as a string (Good for logging) ---
Traceback (most recent call last):
  File "C:\Users\danny.hur\Desktop\traceback-test.py", line 11, in <module>
    run_process()
    ~~~~~~~~~~~^^
  File "C:\Users\danny.hur\Desktop\traceback-test.py", line 8, in run_process
    cause_error()
    ~~~~~~~~~~~^^
  File "C:\Users\danny.hur\Desktop\traceback-test.py", line 5, in cause_error
    return 1 / 0
           ~~^~~
ZeroDivisionError: division by zero


--- 3. Extract structured frames ---
File: C:\Users\danny.hur\Desktop\traceback-test.py, Line: 11, In: <module>
File: C:\Users\danny.hur\Desktop\traceback-test.py, Line: 8, In: run_process
File: C:\Users\danny.hur\Desktop\traceback-test.py, Line: 5, In: cause_error
```