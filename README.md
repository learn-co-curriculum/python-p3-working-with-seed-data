# Working with Seed Data

## Learning Goals

- Use an external library to simplify tasks from ORM and Advanced ORM.
- Manage database tables and schemas without ever writing SQL through Alembic.
- Use SQLAlchemy to create, read, update and delete records in a SQL database.

***

## Key Vocab

- **Persist**: save a schema in a database.
- **Migration**: the process of moving data from one or more databases to one
  or more target databases.
- **Seed**: to fill a database with an initial set of data.

***

## Introduction

What good is a database without any data? When working with any application
involving a database, it's a good idea to populate your database with some
realistic data when you are working on building new features. SQLAlchemy, and
many other ORMs, refer to the process of adding sample data to the database as
**"seeding"** the database. In this lesson, we'll see some of the conventions
and built-in features that make it easy to seed data in an SQLAlchemy
application.

This lesson is set up as a code-along, so make sure to fork and clone the
lesson. Then run these commands to set up the dependencies and set up the
database:

```console
$ pipenv install && pipenv shell

# => Installing dependencies from Pipfile.lock (xxxxxx)...
# =>   🐍   ▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉ X/X — XX:XX:XX
# => To activate this projects virtualenv, run pipenv shell.
# => Alternatively, run a command inside the virtualenv with pipenv run.
# => Launching subshell in virtual environment...
# =>  . /python-p3-working-with-seed-data/.venv/bin/activate
# => $  . /python-p3-working-with-seed-data/.venv/bin/activate

$ cd seed_db && alembic upgrade head
```

In this application, we have two migrations: one for our declarative `Base`,
and a second for a table called `games`.

```py
# app//db.py

class Game(Base):
    __tablename__ = 'games'

    id = Column(Integer(), primary_key=True)
    title = Column(String())
    genre = Column(String())
    platform = Column(String())
    price = Column(Integer())
    created_at = Column(DateTime(), server_default=func.now())
    updated_at = Column(DateTime(), onupdate=func.now())

```

***

## Why Do We Need Seed Data?

With SQLAlchemy, we've seen how simple it is to add data to a database by
using built-in methods that will write SQL code for us. For instance, to create
a new record in the `games` table, you can open up the Python shell, generate
a SQLAlchemy session, create an instance of the `Game` model, and commit it to
the session. To simplify this even further, we've used `app/debug.py` to create
a session and `import` relevant classes. Run `debug.py` from the `seed_db` and
directory and enter the following into the `ipdb` shell:

```py
botw = Game(title="Breath of the Wild", platform="Switch", genre="Adventure", price=60)
session.add(botw)
session.commit()
```

Awesome! Our database now has some data in it. We can create a few more games:

```py
ffvii = Game(title="Final Fantasy VII", platform="Playstation", genre="RPG", price=30)
mk8 = Game(title="Mario Kart 8", platform="Switch", genre="Racing", price=50)
session.bulk_save_objects([ffvii, mk8])
session.commit()
```

<details>
  <summary>
    <em><code>bulk_save_objects</code> is a useful method, but it doesn't
        carry out all the same tasks as <code>add</code>. What attributes will
        <code>botw</code> have that <code>ffvii</code> and <code>mk8</code>
        are still missing?</em>
  </summary>

  <h3><code>id</code> and timestamps.</h3>
  <p><code>bulk_save_objects</code> does not return any new data from the
     records it creates.</p>
  <p>You can set the keyword argument <code>return_defaults=True</code> to get
     IDs and other automatically assigned attributes, but this requires
     SQLAlchemy to execute statements for each record individually.</p>
  <p>Make sure to think about the features that you need before you pick which
     CRUD methods to use!</p>
</details>
<br/>

Since these records are saved in the database rather than in Python's memory, we
know that even after we exit the `ipdb` shell, we'll still be able to retrieve
this data.

But how can we share this data with other developers who are working on the same
application? How could we recover this data if our development database was
deleted? We could include the database in version control, but this is generally
considered bad practice: since our database might get quite large over time,
it's not practical to include it in version control (you'll even notice that in
our SQLAlchemy projects' `.gitignore` file, we include a line that instructs
Git not to track any `.sqlite3` files). There's got to be a better way!

The common approach to this problem is that instead of sharing the actual
database with other developers, we share the **instructions for creating data in
the database** with other developers. By convention, the way we do this is by
creating a Python file, `app/seed.py`, which is used to populate our database.

We've already seen a similar scenario by which we can share instructions for
setting up a database with other developers: using SQLAlchemy migrations to
define how the database tables should look. Now, we'll have two kinds of database
instructions we can use:

- Migrations: define how our tables should be set up.
- Seeds: add data to those tables.

***

## Using the `seed.py` File

To use the `seed.py` file to add data to the database, all we need to do is
write code that uses SQLAlchemy methods to create new records. Add this to
the `app/seed.py` file below the creation of the `session` object:

```py
# seed_db/app/seed.py

...

botw = Game(title="Breath of the Wild", platform="Switch", genre="Adventure", price=60)
ffvii = Game(title="Final Fantasy VII", platform="Playstation", genre="RPG", price=30)
mk8 = Game(title="Mario Kart 8", platform="Switch", genre="Racing", price=50)

session.bulk_save_objects([botw, ffvii, mk8])
session.commit()
```

To run this code, simply run `python db/seed.py`. You should not see any output
if it executes without error.

Run the `debug.py` script again to enter an `ipdb` shell.

```py
session.query(Game).count()
# => 3
session.query(Game)[-1]
# => Game(id=3, title="Mario Kart 8", platform="Switch)"
```

Awesome! Exit out of the console.

What happens if we want to add some more data to the database? Well, we could
try adding another `add` call in our `app/seed.py` file:

```py
# py/seed.py

botw = Game(title="Breath of the Wild", platform="Switch", genre="Adventure", price=60)
ffvii = Game(title="Final Fantasy VII", platform="Playstation", genre="RPG", price=30)
mk8 = Game(title="Mario Kart 8", platform="Switch", genre="Racing", price=50)
ccs = Game(title="Candy Crush Saga", platform="Mobile", genre="Puzzle", price=0)
```

Run the seed file again, and let's see our updated data:

```py
session.query(Game)[-1]
# => Game(id=7, title="Candy Crush Saga", platform="Mobile)"
session.query(Game).count()
# => 7
```

Hmm, we only added four games in the `db/seed.py` file: why are there now seven
games in the database? Well, remember — every time we run `app/seed.py`, we are
creating **new** records in the `games` table. There's nothing stopping our code
from producing duplicate data in the database. We're just instructing SQLAlchemy
to create new code using this file!

We can modify our seed file to clear our database before each new seed to avoid
this complication in the future. Add these commands right after the
instantiation of your `session` (but before the creation of any new records):

```py
session.query(Game).delete()
session.commit()
```

These commands remove the data from the `games` table and then re-run the
seed file. It's handy if you want to start fresh! Just be cautious factoring
this into a seed file, because it will inevitably remove all of your data.

We can now see our fresh database with just four records in the `games` table, as
intended. Run `python debug.py`:

```py
session.query(Game).count()
# => 4
```

***

## Generating Randomized Data

One challenge of seeding a database is thinking up lots of sample data.
Ultimately, when you're developing an application, it's helpful to have
realistic data, but the actual content is not so important.

One tool that can be used to help generate a lot of realistic randomized data is
the [Faker library][faker]. This library is already included in the Pipfile for
this application, so we can try it out. Run `python debug.py`, and try
out some Faker methods:

```py
fake = Faker()
fake.name()
# => 'Samantha Taylor'
fake.name()
# => 'Connie Ferguson'
fake.name()
# => 'Christopher Ortega'
```

As you can see, every time we call the `name` method, we get a new random name.
Faker has a lot of [built-in randomized data generators][faker] that you can use:

```py
fake.email()
# =>'hubbardpatricia@example.com'
fake.color()
# => '#7148af'
fake.profile()
# => {'job': 'Garment/textile technologist', 'company': 'Jones PLC', 'ssn': '768-52-6547', 'residence': '7571 Michael Coves\nNorth Daniel, VA 39350', 'current_location': (Decimal('-30.883927'), Decimal('65.589098')), 'blood_group': 'O-', 'website': ['http://www.grimes.org/', 'http://sanders.net/', 'https://manning-cowan.info/', 'https://www.sims-smith.info/'], 'username': 'julia98', 'name': 'Jillian Morris', 'sex': 'F', 'address': 'USNS Brown\nFPO AA 04021', 'mail': 'james05@yahoo.com', 'birthdate': datetime.date(1990, 6, 20)}
```

Let's use Faker to generate 50 random games (we will use the random library to
generate prices). Replace the data after our data deletion in the `seed.py`
file with the following code:

```py
# Add a console message so we can see output when the seed file runs
print("Seeding games...")

games = [
    Game(
        title=fake.name(),
        genre=fake.word(),
        platform=fake.word(),
        price=random.randint(0, 60)
    )
for i in range(50)]

session.bulk_save_objects(games)
session.commit()
```

<details>
  <summary>
    <em>What do we call the syntax that we used to create the <code>games</code>
        variable?</em>
  </summary>

  <h3>List Interpretation.</h3>
  <p>Interpretations allow us to create lists without using loops. They're a
     powerful staple of Python code, so make sure you don't forget!</p>
</details>
<br/>

Then, run `python app/seed.py` to reseed the database:

```py
session.query(Game).count()
# => 50
session.query(Game).first()
# => Game(id=1, title="Lisa Barton", platform="hotel)"
session.query(Game)[-1]
# => Game(id=50, title="Jesus Anderson", platform="meeting)"
```

Great! Now we've got plenty of seed data to work with, and an easy way for
ourselves or other developers to populate the database any time we need to do
so.

***

## Conclusion

In this lesson, we learned the importance of having a seed file along with our
database migrations in order for ourselves and other developers to quickly set
up the database with sample data. We also learned how to use the Faker library
to quickly generate randomized seed data.

***

## Resources

- [SQLAlchemy ORM Documentation][sqlaorm]
- [Faker Documentation][faker]
- [random — Generate pseudo-random numbers - Python](https://docs.python.org/3/library/random.html)

[faker]: https://faker.readthedocs.io/en/master/index.html]
[sqlaorm]: https://docs.sqlalchemy.org/en/14/orm/
