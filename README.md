## Initial Setup

- Clone this repo
- We will write the `app.py` file together

## After the in-class demo
- `fly launch` to create new app and **select yes for creating with postgres db**
  - **Add payment method if necessary, just make sure you don't exceed 3 VMs at once**
  - Make sure to choose the `dev` instance of the DB
  - Something like this will be printed to your console. **SAVE IT** for later.

```
Postgres cluster autumn-cherry-7687 created
Username: postgres
Password: XfBn9F9x4gfqrVF
Hostname: autumn-cherry-7687.internal
Proxy port: 5432
Postgres port: 5433
Connection string: postgres://postgres:XfBn9F9x4gfqrVF@autumn-cherry-7687.internal:5432
```

- If database creation fails or times out, try running `fly agent restart`
- If you accidentally select no when creating the database then
  - `fly pg create`
  - `fly pg attach --app <app-name> <postgres-app-name>`
- Test that you can connect to your DB with `fly pg connect -a <postgres-app-name>`
  - You should expect to connect to a postgres DB
- Create and activate virtual environment if you haven't already with
  - `python3 -m venv venv`
  - `. venv/bin/activate`
- Install sqlalchemy and psycopg2
  - Note: On M1 mac install `psycopg2-binary` instead (or if you run into other issues?)
  - pip freeze > requirements.txt to let fly.io know it needs to install new dependencies

## SQLAlchemy Setup

- Note: fly outputs a connection string that is incompatible with SQLAlchemy. For SQLAlchemy we need to change postgres -> postgresql in connection string https://stackoverflow.com/a/64698899/3772221
  - To do this we will change the `DATABASE_URL` secret to reflect this with `fly secrets set DATABASE_URL=postgresql://<the rest of your connection string>`
    - So **as an example** (this will not work for you if you just copy this) given that the above connection string is `postgres://postgres:XfBn9F9x4gfqrVF@autumn-cherry-7687.internal:5432` all we will be doing is adding a "ql" to the end of the first part so you have `postgresql://postgres:XfBn9F9x4gfqrVF@autumn-cherry-7687.internal:5432`
- The example here is based on https://flask-sqlalchemy.palletsprojects.com/en/2.x/quickstart/
  - IMPORTANT: `user` is a reserved table name in PostgresDB so for the "A Minimal Application" example, change the model name from `class User` -> `class Person`
  - When connecting to your SQLAlchemy DB set the connection string to be the `DATABASE_URL` environment variable
    - `app.config['SQLALCHEMY_DATABASE_URI'] = os.getenv('DATABASE_URL')`
- Put `db.create_all()` under `with app.app_context():` https://stackoverflow.com/a/34123343/3772221

## Local Development

- To enable local development, proxy your production DB to localhost with
  `flyctl proxy 5432 -a <postgres-app-name>` (run this in a separate terminal since it needs to stay active)
  - If you get `Error listen tcp 127.0.0.1:5432: bind: address already in use` use another port
    like this `flyctl proxy 15433:5432 -a <postgres-app-name>`, which binds to the first number `15433`
- Run your flask app with the `DATABASE_URL` env variable set to the proxied postgresql connection string. **important** this won't work if you didn't follow the above step
  - This means instead of something like `@autumn-cherry-7687.internal:5432` at the end of your
    string, instead you'll have `@localhost:5432`. Remember that if you changed the port, it'd be `@localhost:15433` or whatever else you set it to.
  - So to locally develop you do `FLASK_APP=<your app> DATABASE_URL=<your local connection string>`
    - For me, this was: `FLASK_APP=app DATABASE_URL=postgresql://postgres:XfBn9F9x4gfqrVF@localhost:15433 flask run`
- Run your app locally
- Connect to your postgresql DB again using `fly pg connect -a <name>`
  - Run `\d` to see if the table `person` exists
  - Add a person into the table:
    - `INSERT INTO person (username, email) VALUES ('a', 'a@f.com');`

## Check production

- `fly deploy`
- `fly open`

## Useful Docs

- Flask setup https://flask.palletsprojects.com/en/2.2.x/installation/
- https://fly.io/docs/postgres/the-basics/connecting/

---

## **FAQ**

## I want to change the schema of one of my tables. Can I?
If you try to add/delete columns from a table (i.e. change the class variables of one of your database models) after the table has been created, your database usually won’t update to reflect your changes. This is because the `db.create_all()` call in SQLAlchemy first checks if each table exists, and then creates any tables that don’t. If you change the schema, it won’t realize that it needs to add columns to your table the next time it runs.

Your best bet is to delete your table and start over. To do that, connect to your table as in the demo above. In the prompt, enter `\d` to see a list of your tables. Then, enter `DROP TABLE “<table>”;`, where `<table>` is your table name.

(Postgresql strings are case-sensitive if double quotes are used, and case-insensitive otherwise)

## How should I get started?
Everyone will have a slightly different way of fitting together the different concepts in this app, so there’s no single solution to this question. But to get you started, here’s a rough list of the tasks that need to be done:
Add a form to your main page where users can enter ratings and comments.
Add a database to your app where comments and ratings can be stored. 
Add a page where users can sign up for and log in to the app.
Once you’ve figured out the login logic, change your main page logic so that it displays all comments associated with the current movie.

This project shouldn’t be a huge number of lines of code, but it’s tricky to get started because you have to take distinct concepts -- forms, databases, login -- and fit them all together. It’s completely normal and even expected to be unsure of how to start, and this is actually a common situation in software engineering.

One way to get out of analysis paralysis is to **just build something**. Take whatever part of the app makes sense to you, whether it’s adding a DB, adding a login page, or whatnot, and build that, and see what happens. More often than not, you’ll see the next step you have to take (“I built this form, so now I need some server logic to receive the data”; “I added a database, so now I need to figure out what data I want to store in it”) once you force yourself to start moving. 

## What’s this UserMixin thing in the Flask-Login docs? How do I use it?
Flask-Login will keep track of an object representing the current user of your app. [This section of the docs](https://flask-login.readthedocs.io/en/latest/#your-user-class) describes how Flask expects that object to look. In particular, Flask wants you to create a user class and then pass an instance of it to `login_user()` when you want to log that user in. 

Flask-Login provides a UserMixin class that will implement most of those requirements up front. So when you define your user class, you can do this to inherit all those attributes:

```
class AppPerson(UserMixin):
```

Note that this is not the same class you're using as one of your database models, if you're keeping a table of users, e.g.
  
```
class DBPerson(db.Model):
```
  
You'll have to figure out how to translate an `AppPerson` into a `DBPerson` and vice versa.

## How do we access whoever is logged in within python?
```
from flask_login import current_user
```
`current_user` from the `flask-login` library will let you access the user variable within the python file. Note that this will return EXACTLY the object you passed to `login_user()`, whenever you called that function.
