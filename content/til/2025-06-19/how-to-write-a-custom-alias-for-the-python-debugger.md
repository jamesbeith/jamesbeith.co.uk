---
title: "How to write a custom alias for the Python debugger"
date: "2025-06-19T16:45:00+10:00"
---

Recently shared at [work](https://kraken.tech/) was this TIL from Samuel, [Custom alias for pretty printing in Python debugger with .pdbrc (including Django models!)](https://www.samuelliedtke.com/blog/til-rich-pretty-print-in-python-debugger-with-pdbrc).

Here’s my ever so slightly adapted version which also includes printing a reminder to me that these exist.

```text
alias rp import rich; rich.print(%*)
alias rpo import rich; from django import forms; rich.print(forms.model_to_dict(%*))

print("Aliases: Use `rp obj` and `rpo instance` to print objects and model instances.\n")
```

I already had something similar in my [IPython](https://github.com/ipython/ipython) startup script using [pprint](https://docs.python.org/3/library/pprint.html) but I’ve now adapted it to use [Rich](https://github.com/Textualize/rich) too.

```python
from django import forms

import rich

rp = rich.print


def rpo(*args):
    for instance in args:
        rich.print(forms.model_to_dict(instance))


print("Helpers: Use `rp(*args)` and `rpo(*args)` to print objects and model instances.\n")
```

Thanks for sharing, Samuel.
