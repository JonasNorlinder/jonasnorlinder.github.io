---
layout: post
title: Programmatically Checking an Array in GDB
---
Following my previous adventures in using ChatGPT to produce scripts to become more efficient while debugging OpenJDK, here is another neat one to find unexpected values in an array. This time it got it right the first time 🤖.

```python
class CheckDoneArray(gdb.Command):
    def __init__(self):
        super(CheckDoneArray, self).__init__("check_done_array", gdb.COMMAND_USER)

    def invoke(self, arg, from_tty):
        try:
            size = int(arg)
        except ValueError:
            print("Invalid argument. Please provide the size of the array.")
            return

        for i in range(size):
            done_value = gdb.parse_and_eval(
                f"ZFromSpacePool::_pool->_array[{i}]->_done")
            if not done_value:
                print(f"Element {i}: _done = {done_value}")

CheckDoneArray()
```

To use it, store the code as `check_array.py` and then type `source check_array.py`. If you always want it to be loaded, then you can put that line in your `.gdbinit` file.