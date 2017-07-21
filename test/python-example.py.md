# Python Example
This is an example of literate programming using python.

Code block without fencing:

    #!/usr/bin/env python

Fenced code block without language:
```

# To check that the generated code is correct run:
# $ litparse -p test/python-example.py.md | diff test/python-example.py -

print "Hello",

```

Fenced code block with language:
```python
# To check that the output is correct run:
# $ litparse test/python-example.py.md

print "World"
```

The output should be:
> Hello World
