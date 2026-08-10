
`Popen().communicate()` is used with Python’s `subprocess` module to interact with a child process.

It:

1. optionally sends input to the process
2. reads everything from the process’s `stdout`
3. reads everything from the process’s `stderr`
4. **waits for the process to finish**
5. returns the captured output

## Basic example

```python
import subprocess

p = subprocess.Popen(
    ["ls", "-l"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE
)

stdout_data, stderr_data = p.communicate()
```

After this:

```python
stdout_data
```

contains what the command printed to standard output.

```python
stderr_data
```

contains what the command printed to standard error.


## What it returns

`communicate()` returns a tuple:

```python
(stdout, stderr)
```

Example:

```python
out, err = p.communicate()
```

If the process output was captured as bytes, you get bytes:

```python
b'file1.txt\nfile2.txt\n'
```

If you used text mode, you get strings:

```python
p = subprocess.Popen(
    ["ls", "-l"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True
)

out, err = p.communicate()
```

Then:

```python
out
```

might be:

```text
file1.txt
file2.txt
```

---

## It waits for the process to finish

Calling:

```python
p.communicate()
```

blocks until the subprocess exits.

So this:

```python
p = subprocess.Popen(["sleep", "5"])
p.communicate()
print("done")
```

will wait about 5 seconds before printing:

```text
done
```

---

## Sending input to the process

You can also use `communicate()` to send data to the process’s standard input.

Example:

```python
p = subprocess.Popen(
    ["grep", "hello"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True
)

out, err = p.communicate("hello world\nbye world\n")
print(out)
```

Output:

```text
hello world
```

Here, `"hello world\nbye world\n"` is sent to the subprocess as input.

---

## Why use `communicate()`?

It is safer than manually doing:

```python
p.stdout.read()
p.stderr.read()
```

because manually reading from one pipe can cause deadlocks if the other pipe fills up.

`communicate()` handles reading and writing in a coordinated way.

---

## Checking the return code

After `communicate()` finishes, you can check:

```python
p.returncode
```

Example:

```python
p = subprocess.Popen(
    ["ls", "/does/not/exist"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True
)

out, err = p.communicate()

print("stdout:", out)
print("stderr:", err)
print("return code:", p.returncode)
```

Possible output:

```text
stdout: 
stderr: ls: cannot access '/does/not/exist': No such file or directory
return code: 2
```

---

## With timeout

You can give it a timeout:

```python
try:
    out, err = p.communicate(timeout=3)
except subprocess.TimeoutExpired:
    p.kill()
    out, err = p.communicate()
```

If the process does not finish within 3 seconds, Python raises `subprocess.TimeoutExpired`.

---

## Summary

```python
out, err = p.communicate()
```

means:

> Wait for the process to finish, collect its stdout and stderr, and return them.

It is commonly used like this:

```python
import subprocess

p = subprocess.Popen(
    ["some_command"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True
)

stdout, stderr = p.communicate()
return_code = p.returncode
```
