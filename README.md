## Initial Setup

- `fly launch` to create new app and **select yes for creating with postgres db**
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
- This repo is based on https://fly.io/docs/languages-and-frameworks/python/
  - The only difference is I changed hellofly.py -> app.py and made necessary changes after that
- https://fly.io/docs/postgres/the-basics/connecting/
