---
title: "Flask App for LLM Summarizer (Part 1)"
date: 2025-05-27
permalink: /posts/2025-05-27-llm_summarizer_pt1/
excerpt: In part one (one-sentence summary)...
tags:
  - Flask
  - Huggingface
  - sqlite
---
Note: the structure of this app is inspired by the [tutorial](https://flask.palletsprojects.com/en/stable/tutorial/) on Flask’s main documentation page.

Traditional data scientists tend to spend the bulk of their time doing model R&D in Jupyter notebooks, text editors, and the command line. However, there comes a time in data scientists’ careers where they have to demo a model to people who have no idea what any of those tools are. The last thing one should do in this situation is show them a bunch of code that looks like gibberish to their eyes. They want to see a product that gets to the point -- what cool thing can this do for me? Luckily, there are plenty of lightweight web application frameworks designed to make demoing models via the browser easy, with only a basic level of html and css expertise necessary. 

In this post I’ll focus specifically on the Flask framework to host a HuggingFace document summarization large language model. The app contains a text box where users can paste in text and receive a ~1-paragraph summary of its content. A sqlite database saves the prompts and summaries so that history can be tracked across the session and cleared by the user as needed. 

First, let’s lay out the file structure for the project:

```
summarizer-webpage/
|-- llm_app/
|   |-- __init__.py         # Flask knows to run create_app() at initialization
|   |-- db.py               # DB connection and init functions
|   |-- schema.sql          # SQL script to create DB schema
|   |-- summarizer.py       # Blueprint containing application routes
|   |-- static/
|       |-- style.css       # color and chat box style config
|   |-- templates/
|       |-- form.html       # dynamic html for the webpage
|
|-- instance/               # Flask's special instance folder (private)
|   |-- config.py           # Contains secret key and other instance-specific config
|   |-- llm_sumry.sqlite    # SQLite DB file
```

The `__init__.py` file employs a Flask design pattern called an **application factory**, in which the app is created inside a function. Instead of defining the app at the top of the script like so:

```
app = Flask(__name__)
```

one would do this:
```python
def create_app(config=None):
    app = Flask(__name__)
    # config, blueprints, extensions, etc.
    return app
```

Flask will automatically detect the factory if it is named `create_app` or `make_app` and run the function when a user enters `flask --app [app_name] run` (`app_name` would be `llm_app` in this project). Here the factory is initializing the app and setting up config values:

```python
import os
from flask import Flask
from . import db

def create_app(test_config=None):
    '''
    - Creates and configures the Flask app.
    - Loads default and optional test configuration.
    - Ensures the instance folder exists.
    - Registers the summarizer blueprint and initializes the database.

    Args:
        test_config (dict, optional): Configuration dictionary used during testing.
                                       Overrides default and file-based settings.

    Returns:
        Flask: The configured Flask application.
    '''
    app = Flask(__name__)
    
    app.config.from_mapping(
    SECRET_KEY="dev",   # will get replaced with secret in conf if not testing
    DATABASE=os.path.join(app.instance_path, "llm_sumry.sqlite"),
    MAX_LEN=200,       # the rest of these are params for LLM model
    MIN_LEN=30)

    if test_config:
        # running tests, pass test config in
        app.config.from_mapping(test_config)
    else:
        # pass actual config in (contains secret key)
        #--------------------------------------------------------#
        # run python -c 'import secrets; print(secrets.token_hex())'
        # and paste this into config.py: SECRET_KEY=<output_of_above>
        app.config.from_pyfile("config.py", silent=True)
    
    # Make an instance folder if it doesn't exist. If it does, try/except avoids crashing the app
    try:
        os.makedirs(app.instance_path)
    except OSError:
        pass

    return app
```

- `app = Flask(__name__)` creates a Flask app instance, which is the central object that handles requests, config, and more. 
- `app.config.from_mapping` sets up config values within the app instance that get used later on. 
- the `app.instance_path` param inside `DATABASE=os.path.join(app.instance_path, "llm_sumry.sqlite")` is a special path used by Flask for instance-specific files that shouldn't be committed to version control, such as secrets or databases. 
- `app.config.from_mapping(test_config)` is helpful when running unit tests -- test config values can be passed in that override the default values assigned earlier. 
- `app.config.from_pyfile("config.py", silent=True)` assigns config values from config.py (silent param ignores errors). This is helpful when the user has a true secret key that shouldn't be hard-coded into the script. Flask looks for this `config.py` file in the `app.instance_path` mentioned above. 

---

Now we need to set up the SQLite database that stores user prompts and LLM summaries. The functionality for this lives in `db.py`:

```python
import sqlite3
import click
from flask import current_app, g

def get_db():
	'''
    Retrieve the SQLite DB connection stored in Flask's `g` 
    context variable. If no connection exists, open one and store it.
    '''
	if 'db' not in g:
		g.db = sqlite3.connect(
			current_app.config['DATABASE'],
			detect_types=sqlite3.PARSE_DECLTYPES
		)
		g.db.row_factory = sqlite3.Row
	return g.db


def close_db(e=None):
	'''
	Close the DB connection at the end of the request, if it exists.
	'''
	db = g.pop('db', None)
	if db is not None:
		db.close()


def init_db():
	'''
    Initialize the DB using schema.sql.
    This drops and recreates tables as defined in the SQL script.
    '''
	db = get_db()
	with current_app.open_resource("schema.sql") as f:
		db.executescript(f.read().decode("utf-8"))


@click.command("init-db")
def init_db_command():
	'''
    Flask CLI command to initialize the DB by running schema.sql.
    Run `flask --app llm_app init-db` the first time you use the app.
    '''
	init_db()
	click.echo("Initialized the database")


def init_app(app):
	'''
    Register DB-related functions with the app:
    	- Ensure the DB connection is closed after each request.
    	- Add CLI command for DB initialization.
    '''
	app.teardown_appcontext(close_db)
	app.cli.add_command(init_db_command)

```

