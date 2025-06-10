---
title: "Easily Locate Local Source Code for Python Modules"
date: 2025-06-10
permalink: /posts/2025-06-10-find_mod_path/
excerpt: How to build a script to locate and classify Python modules, showing practical use cases and step-by-step improvements. 
tags:
---

When working with Python modules, one of the first things I do is google the documentation to understand what their modules and methods do. Oftentimes, this along with some experimentation is enough for my needs. If I want to go deeper, I'll google the source code, which is usually accessible via the module's GitHub page. 

However, there are lots of good reasons to read through the actual source code installed on your machine. For instance, the online source code may be a different version than the code your machine is running (e.g., maybe you installed the latest version a year ago and three updates have happened since then). Also, you can inspect the source code on your machine without an internet connection and even locally experiment by tweaking functionality on certain lines.

Sometimes though, it can be a pain to find where this code lives. It could live in your Python site-packages directory or somewhere in your anaconda install. For example, you could have multiple versions of the same `requests` library installed and maybe aren't sure which version is running. An easy way to find the source code for `requests` that Python is running is to enter the following on the terminal:

```bash
$ python -c "import requests; print(requests.__file__)"

/opt/anaconda3/lib/python3.12/site-packages/requests/__init__.py
```

The `-c` option in Python runs the following quote as a sort of mini Python script. `__file__` is a built-in variable in Python that holds the location path of the current file being executed. Instead of typing that out every time, I am lazy so will make it into a script called `find_mod_v1.py`:

```python
import importlib
import sys

def find_mod_path(mod_name):
	"""
    Imports a module and prints its file path, if available.
    """
		
	print(f"\n------------ Module: {mod_name} ------------")
	try:
		module = importlib.import_module(mod_name)
	
	# Handle case where the module doesn't exist
	except ModuleNotFoundError:
		print(f"Module '{mod_name}' not found\n")
		return
      
	# Handle other unexpected import errors
	except Exception as e:
		print(f"Error importing module '{mod_name}': {e}\n")
		return

	print(f"Module '{mod_name}' is located at:\n{module.__file__}\n")


if __name__ == "__main__":

	# Check that exactly one module name was provided
	if len(sys.argv) != 2:
		print("Usage: python find_mod_path <module_name>")
		sys.exit(1)

	# Call the main function with the provided module name
	find_mod_path(sys.argv[1])
```

Now, instead of the long previous command, I can simply run the script like so:
```bash
$ python find_mod_v1.py requests

------------ Module: requests ------------
Module 'requests' is located at:
/opt/anaconda3/lib/python3.12/site-packages/requests/__init__.py

$ python find_mod_v1.py token

------------ Module: token ------------
Module 'token' is located at:
/opt/anaconda3/lib/python3.12/token.py

```

Notice how `token.__file__` is an actual file called `token.py`, but `requests.__file__` equates to an `__init__.py` file inside a `requests` directory. Why are their structures different? While `token` is a simple module whose functionality is contained in a single file called `token.py`, `requests` is a special type of module called a package, which is a directory containing an `__init__.py` file along with other files that contain functionality related to `requests`.

This basic script appears to work decently well, but there are cases it's not prepared for. For instance, see what happens when running the script with the Python `sys` module as an argument:

```bash
$ python find_mod_v1.py sys

------------ Module: sys ------------
Traceback (most recent call last):
  File "/Users/gcano01/projects/convenience/find_mod_v1.py", line 23, in <module>
    find_mod_path(sys.argv[1])
  File "/Users/gcano01/projects/convenience/find_mod_v1.py", line 17, in find_mod_path
    print(f"Module '{mod_name}' is located at:\n{module.__file__}\n")
                                                 ^^^^^^^^^^^^^^^
AttributeError: module 'sys' has no attribute '__file__'. Did you mean: '__name__'?
```

`sys` is a Python module that can be imported, but it is a **built-in module**, meaning its code is compiled within the Python interpreter itself and can't be viewed directly on your machine (the source code in C for `sys` can be found [here](https://github.com/python/cpython/blob/main/Python/sysmodule.c)). This case needs to be acknowledged in the script. 

Also, while running the script on random module arguments, I noticed this odd file extension for the `math` module:

```bash
$ python find_mod_v1.py math

------------ Module: math ------------
Module 'math' is located at:
/opt/anaconda3/lib/python3.12/lib-dynload/math.cpython-312-darwin.so

```

It turns out, `.so` stands for shared object, which is a compiled binary typically from a C or C++ script. This compiled code runs much faster than a `py` file for tasks like numerical computation, which helps explain why a module like `math` would list that as its source. To find the human-readable code for `math`, you'd have to go online [here](https://github.com/python/cpython/blob/main/Modules/mathmodule.c).

Due to all these categories I've been running into (simple module, package, built-in module, `.so` extension), I'll add an option to categorize in addition to providing the `__file__`:

```python
import importlib
import os
import argparse

def find_mod_path(mod_name, classify_file=False):
	"""
	Prints file location of a given module 
	and classifies its type if requested.
	"""

	print(f"\n------------ Module: {mod_name} ------------")
	try:
		module = importlib.import_module(mod_name)
	
	# Handle case where module doesn't exist
	except ModuleNotFoundError:
		print(f"Module '{mod_name}' not found\n")
		return
      
	# Catch any other import-related errors
	except Exception as e:
		print(f"Error importing module '{mod_name}': {e}\n")
		return

	if hasattr(module, "__file__") and module.__file__:
		print(f"Module '{mod_name}' is located at:\n{module.__file__}\n")
	else:
		# Built-in modules will not have a __file__
		print(f"Module '{mod_name}' is built-in (no file path, human-readable code online)\n")
		return
    
	if classify_file:
		path = module.__file__
		if os.path.basename(path) == "__init__.py":
			print("File is a package (directory with __init__.py).\n")
		
        # .pyd is essentially the Windows version of .so
		elif path.endswith(('.so', '.pyd')):
			print("File is a C extension module, human-readable code online.\n")
			
		elif path.endswith(".py"):
			print("File is a pure python module (.py file)\n")
			
		else:
			print("File type not classified by this program (yet).\n")
			

if __name__ == "__main__":
	parser = argparse.ArgumentParser(description="Finds the file path of a \
										python module or package.")

	parser.add_argument("mod_name", help="Name of the module to inspect")

	# Optional flag to include classification details
	parser.add_argument("--details", action="store_true", 
					    dest="classify_file", help="Show extra info about module")
	
	args = parser.parse_args()
	find_mod_path(args.mod_name, args.classify_file)
```

In version 2 of the script, I replaced the use of `sys` with the more robust `argparse` module. The script now takes an optional `--details` argument. The `action="store_true"` and `dest="classify_file"` params tell argparse that if `--details` is included, store `True` in `classify_file` and store `False` if it is omitted. Now, if the `classify_file` flag is set in `find_mod_path()`, a new statement describing the type of file is printed. 

Here is the new script being run:

```bash
$ python find_mod_v2.py requests --details

------------ Module: requests ------------
Module 'requests' is located at:
/opt/anaconda3/lib/python3.12/site-packages/requests/__init__.py

File is a package (directory with __init__.py).

$ python find_mod_v2.py token --details

------------ Module: token ------------
Module 'token' is located at:
/opt/anaconda3/lib/python3.12/token.py

File is a pure python module (.py file)

$ python find_mod_v2.py sys --details

------------ Module: sys ------------
Module 'sys' is built-in (no file path, human-readable code online)

$ python find_mod_v2.py math --details

------------ Module: math ------------
Module 'math' is located at:
/opt/anaconda3/lib/python3.12/lib-dynload/math.cpython-312-darwin.so

File is a C extension module, human-readable code online.

```

One more improvement I could think of is the ability to process a list of modules, which is a fairly simple modification involving a tweak to `argparser` and a `for` loop:

```python
import importlib
import os
import argparse

def find_mod_path(mod_name, classify_file=False):
	"""
	Prints file location of a given module 
	and classifies its type if requested.
	"""

	print(f"\n------------ Module: {mod_name} ------------")
	try:
		module = importlib.import_module(mod_name)
	
	# Handle case where module doesn't exist
	except ModuleNotFoundError:
		print(f"Module '{mod_name}' not found\n")
		return
      
	# Catch any other import-related errors
	except Exception as e:
		print(f"Error importing module '{mod_name}': {e}\n")
		return

	if hasattr(module, "__file__") and module.__file__:
		print(f"Module '{mod_name}' is located at:\n{module.__file__}\n")
	else:
		# Built-in modules will not have a __file__
		print(f"Module '{mod_name}' is built-in (no file path, human-readable code online)\n")
		return
    
	if classify_file:
		path = module.__file__
		if os.path.basename(path) == "__init__.py":
			print("File is a package (directory with __init__.py).\n")
		
        # .pyd is essentially the Windows version of .so
		elif path.endswith(('.so', '.pyd')):
			print("File is a C extension module, human-readable code online.\n")
			
		elif path.endswith(".py"):
			print("File is a pure python module (.py file)\n")
			
		else:
			print("File type not classified by this program (yet).\n")


def find_mod_paths(mod_names, classify_file=False):
	"""
	Loop over a list of module names and print 
	their file locations and classifications.
	"""
	for mod_name in mod_names:
		find_mod_path(mod_name, classify_file=classify_file)

if __name__ == "__main__":
	parser = argparse.ArgumentParser(description="Finds the file path of a \
										python module or package.")

	# Accept one or more module names to look up
	parser.add_argument("mod_names", nargs = "+", help="Name of the module to inspect")

	# Optional flag to include classification details
	parser.add_argument("--details", action="store_true", 
					    dest="classify_file", help="Show extra info about modules")
	
	args = parser.parse_args()
	find_mod_paths(args.mod_names, args.classify_file)
```

In the above script, I added the `nargs = "+"` param to the first `add_argument()` method. This param tells `argparser` that `mod_names` should accept a list of one or more args (a `*` instead of a `+` is 0 or more args, `?` is 0 or 1 args, any number `N` is exactly that number of args). So if the user entered `find_mod_v3.py numpy pandas sys`, `args.mod_names` would equal `[numpy, pandas, sys]`. When this is passed into `find_mod_paths()`, it simply loops through that list and runs the original `find_mod_path()` function on each element. Here's a demo run:

```bash
$ python find_mod_v3.py token requests sys math --details

------------ Module: token ------------
Module 'token' is located at:
/opt/anaconda3/lib/python3.12/token.py

File is a pure python module (.py file)


------------ Module: requests ------------
Module 'requests' is located at:
/opt/anaconda3/lib/python3.12/site-packages/requests/__init__.py

File is a package (directory with __init__.py).


------------ Module: sys ------------
Module 'sys' is built-in (no file path, human-readable code online)


------------ Module: math ------------
Module 'math' is located at:
/opt/anaconda3/lib/python3.12/lib-dynload/math.cpython-312-darwin.so

File is a C extension module, human-readable code online.

```

There are other weird types of modules out there that this script can be expanded to cover, such as frozen modules and namespace packages, but the current categorization checks are a start. Another good feature to add on would be reading modules from a file such as `requirements.txt` as opposed to typing out each one manually. One could also add an option to display the installed module version, the list of enhancements could go on and on. For now, I still find this script useful for quickly finding the location of my local source code, and I hope any readers of this blog do as well! 

The full script can be found on my [GitHub page](https://github.com/gabecano4308/find_mod_path/tree/main).