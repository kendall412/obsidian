
In Python, you convert an integer to a byte string using the built-in **`int.to_bytes()`** method. [1](https://www.scaler.com/topics/bytes-python/), [2](https://en.wikiversity.org/wiki/Python_Concepts/Bytes_objects_and_Bytearrays)

Quick Syntax

python

```
integer.to_bytes(length, byteorder, *, signed=False)
```

Use code with caution.

Core Parameters

- **`length`**: The number of bytes to use. A `OverflowError` is raised if the integer does not fit.
- **`byteorder`**: The byte representation order. Use **`'big'`** for most-significant byte first, or **`'little'`** for least-significant byte first.
- **`signed`**: Set to `True` if you are converting a negative number. Defaults to `False`. [1](https://marz.utk.edu/python/binary-files/), [2](https://docs.python.org/3/library/functions.html), [3](https://docs.python.org/3/library/stdtypes.html), [4](https://forums.raspberrypi.com/viewtopic.php?t=197801), [5](https://minimalmodbus.readthedocs.io/en/stable/internalminimalmodbus.html)

Practical Examples

```python
(1024).to_bytes(2, byteorder='big')

# Output: b'\x04\x00'
```

```python
(123456789).to_bytes(4,"little")

# outout: b'\x15\xcd[\x07'
# to get all hex (no ASCII)
b = (123456789).to_bytes(4,"little")
b.hex() = "15cd5b07"
```