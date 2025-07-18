---
title: "Flask App for LLM Summarizer"
date: 2025-05-27
permalink: /posts/2025-05-27-llm_summarizer_pt1/
excerpt: A walkthrough on how to use Flask along with a SQLite database to host a trained model and track interaction history. 
tags:
  - Flask
  - Huggingface
  - sqlite
---
<span style="font-size:13px;">Acknowledgment: the structure of this app is inspired by the [tutorial](https://flask.palletsprojects.com/en/stable/tutorial/) on Flask’s main documentation page.</span>

Traditional data scientists tend to spend the bulk of their time doing model R&D in Jupyter notebooks, text editors, and the command line. However, there comes a time in data scientists’ careers where they have to demo a model to people who have no idea what any of those tools are. The last thing one should do in this situation is show a bunch of code that looks like gibberish to them. They want to see a product that gets to the point -- what cool thing can this do for me? Luckily, there are plenty of lightweight web application frameworks designed to make demoing models via the browser easy, with only a basic level of html and css expertise necessary. 

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

Flask will automatically detect the factory if it is named `create_app` or `make_app` and run the function when a user enters `flask --app [app_name] [command]` (`app_name` would be `llm_app` in this project). Here the factory is initializing the app and setting up config values:

```python
import os
from flask import Flask

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

    from . import db
    # Add init-db CLI command and setup functionality 
    # for closing the db at shutdown
    db.init_app(app)

    from . import summarizer
    # Activate routes for the Blueprint object in summarizer module
    app.register_blueprint(summarizer.bp)

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

`init_app()` is called by `create_app` in the `__init__.py` startup script. Within this function, `app.teardown_appcontext(close_db)` assigns `close_db()` to get run at the end of each request, which closes the db and removes it from `g` (a global variable for storing data during a single app context). `app.cli.add_command(init_db_command)` assigns `init_db_command()` to be run when a user enters `init-db` as a flask command on the command line (i.e., `flask --app llm_app init-db`), see the `@click.command` decorator on top of the function definition. Note that since the db is disconnected after each request, every request needs to begin by running `get_db()` to reconnect with the db. 

When a user runs `init-db` from the command line, a db connection is made and the connection runs the `schema.sql` script, which creates a table for user prompts and LLM responses. This command only needs to be run a single time in the beginning. Shown below is `schema.sql`:

```sql
DROP TABLE IF EXISTS chat_entries;

CREATE TABLE chat_entries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    prompt TEXT NOT NULL,
    response TEXT NOT NULL
);
```

---

The last thing that `create_app()` does before returning is register the summarizer blueprint with the line, `app.register_blueprint(summarizer.bp)`. A Flask blueprint is a way to organize an app into modular components, helpful in case we have multiple families of requests (imagine for example we wanted to add a language translation page to this app, we could make another blueprint for translation routes). The summarizer blueprint lives in `summarizer.py`:

```python
from flask import (
    Blueprint, redirect, render_template, request, url_for, current_app
)
from transformers import pipeline
from . import db


bp = Blueprint('summarizer', __name__)

# Load model once on startup
summarizer = pipeline("summarization", model="facebook/bart-large-cnn")


def summarize(text):
    '''
    Run the text through the pipeline, returning the summary text
    '''
    config = current_app.config

    # run text through summarizer pipeline
    summary = summarizer(text, 
                         max_length=config['MAX_LEN'], 
                         min_length=config['MIN_LEN'], 
                         do_sample=False)

    # return summary
    return summary[0]["summary_text"]


@bp.route("/", methods=["GET", "POST"])
def index():
    '''
    GET: grab chat history from DB and insert into HTML template before rendering.
    POST: same as GET, except add the new prompt/summary to the DB first.
    '''
    llmdb = db.get_db()
    
    if request.method == "POST":

        # grab prompt from request
        prompt = request.form.get("prompt", "")

        # get summary of the prompt
        response = summarize(prompt)

        # add prompt and summary to db
        llmdb.execute(
            'INSERT INTO chat_entries (prompt, response)'
            ' VALUES (?, ?)',
            (prompt, response)
        )
        llmdb.commit()
    
    # grab chat history from DB to insert into the webpage
    chat_history = llmdb.execute(
         'SELECT prompt, response FROM chat_entries ORDER BY id DESC'
    ).fetchall()

    return render_template("form.html", chat_history=chat_history)


@bp.route("/clear", methods=["POST"])
def clear():
    '''
    Delete entries from the chat_entries table and call GET on homepage
    '''
    llmdb = db.get_db()
    llmdb.execute("DELETE FROM chat_entries")
    llmdb.commit()
    return redirect(url_for("summarizer.index"))

```
The summarizer module is imported once by `create_app()`, which runs `summarizer = pipeline("summarization", model="facebook/bart-large-cnn")` at app startup. This line assigns to `summarizer` the Huggingface pipeline that uses the pre-trained bart cnn summarizer model from facebook. This variable will be used by `summarize()` to get a response to the user prompt. 

The app currently only has two routes -- `index` ("/") and `clear` ("/clear"). `index` accepts both `GET` and `POST` requests. At the start of both routes, `get_db()` is called, which establishes a connection to the db. In the `index` `GET` request, the db connection simply queries all the data living in the db and saves it in the `chat_history` variable. Notice in `get_db()`, this line, `g.db.row_factory = sqlite3.Row` ensures the execution output is formatted as a list of dictionaries where each dictionary corresponds to a table row. The data in `chat_history` is embedded into the `form.html` template using `render_template("form.html", chat_history=chat_history)`. 

An `index` `POST` request is similar to an `index` `GET` request, except before the data is queried, the new prompt/summary pair are inserted as a new row into the db. First, the prompt is extracted from the form section of the request (this is the text the user pastes in the chat box before pressing submit, which triggers the `POST` request), then `summarize(prompt)` is called, which runs the text through the Huggingface LLM pipeline before returning the summary output. The rest of the request is then identical to what happens in a `GET` request. 

The `clear` request simply deletes all the data in the database and redirects the user to the homepage, which is effectively a `GET` request to `index`. 

Here is the `form.html` template:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Text Summarizer</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
    <h1>Summarize Text</h1>
    <form method="post">
        <textarea name="prompt" rows="4" cols="80" placeholder="Paste your text here..."></textarea>
        <br>
        <button type="submit">Summarize</button>
    </form>
    <br>
    <form method="post" action="/clear">
        <button type="submit">Clear Chat</button>
    </form>

    <div class="chat-box">
        {% for exchange in chat_history %} 
            <div class="chat-prompt"><strong>You:</strong> {{ exchange.prompt }}</div>
            <div class="chat-response"><strong>Bot:</strong> {{ exchange.response }}</div>
            <hr>
        {% endfor %}
    </div>
</body>
</html>
```

- Within the first `post` form section, the template makes a `textarea` for users to type a prompt into, then the "Summarize" (`submit`) button would trigger a `POST` request including that prompt inside the form.
- Within the second `post` form section, the "Clear Chat" button is paired with the `/clear` route. 
- Within the `chat-box` section, a for loop iterates through the entries in the `chat_history` variable (recall the line `render_template("form.html", chat_history=chat_history)`) and displays the entries as a conversation between "You" and the "Bot".

Before demoing the actual application, here is `style.css`, which is linked to `form.html` and specifies colors, fonts, dimensions, etc. for the webpage:

```css
.chat-box {
    margin-top: 30px;
    width: 100%;
    max-width: 700px;
    background-color: #fff;
    padding: 20px;
    border-radius: 10px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.05);
}

.chat-prompt, .chat-response {
    margin: 10px 0;
}

.chat-prompt {
    color: #333;
    font-weight: bold;
}

.chat-response {
    color: #007BFF;
}
```

---

First we set up a virtual environment and install the necessary libraries while in the `summarizer-webpage` directory:

```bash
conda create --name llm-summarizer python=3.11
conda activate llm-summarizer

pip install flask transformers==4.51.1 numpy==1.25.2 torch==2.2.2
```

Then we initialize the database (this only needs to be run once):
```bash
# should echo, "Initialized the database"
flask --app llm_app init-db
```

Finally, run the application:
```bash
flask --app llm_app run
```

Once running, we can go to `http://127.0.0.1:5000` on the browser and paste some text into the chat-box. It should look something like this:

![image](/images/summarize_text_1.png)

Clicking the Summarize button will send a POST request, which will feed the prompt into the LLM pipeline, store the prompt/summary into SQL, and query all the data in SQL to embed into the webpage. After some load time, here's the new screen:

![image](/images/summarize_text_2.png)

Repeating this process, one can see that the prompt/summary history is maintained across requests by the SQL database:

![image](/images/summarize_text_3.png)

Once the history gets too clogged up, I can press "Clear Chat", which will trigger the `/clear` request and make the screen look the same as it did initially. 

This is a lightweight implementation meant to demonstrate how to host a trained model on a Flask webpage with a sql database. It's not the most barebones way to demo a model on a webpage -- for an even easier way with zero html needed, I recommend checking out [streamlit](https://streamlit.io/). 

All the code for this blog can be found in one place on this [GitHub page](https://github.com/gabecano4308/summarizer-webpage), which also includes a directory for unit tests and a `pyproject.toml` build script. There are plenty of enhancements that can be made to this project, including optimizing for speed and deploying the application on a
cloud platform such as AWS.
