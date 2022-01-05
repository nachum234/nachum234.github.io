---
layout: post
title: "My Python Cheatsheet"
date: 2021-12-06
---

## Tested On

Python: python3.8

## Procedure

* Working with files

```
with open("test.txt", mode='w') as f:
    f.write("my first file\n")
    f.write("Hello World!")
```

```
with open("test.txt", mode='o') as f:
    list = f.readlines()
    for line in list:
        print line
```

* List comprehension
https://docs.python.org/3/tutorial/datastructures.html#list-comprehensions

```
squares = [x**2 for x in range(10)]
```

```
words = ["I", "love", "hummus", "a", "lot"]
short_words = [word for word in words if len(word) <= 3 ] # short_words = ["I", "a", "lot"]
```
