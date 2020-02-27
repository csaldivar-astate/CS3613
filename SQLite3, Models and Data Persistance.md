# Using SQLite3

First install sqlite3 on your Ubuntu VM with these commands:

```
sudo apt update
sudo apt install sqlite3
```

## Useful Commands
```
.exit        # exits
.tables      # shows all tables
.mode column # shows columns in sql query output
.headers on  # shows headers in sql query output
.schema Table_Name # shows the schema for the specified table
```

## SQLite3 config file

I like have columns and headers turned on for my sql queries; however, they're off by default. I use a `.sqliterc` file to turn them on so I don't have to type the commands to turn on columns and headers everytime I use sqlite3. **This is not neccessary so you can skipt this if you don't care.**

Create an `.sqliterc` file using this command

```
nano ~/.sqliterc
```

This opens a command line editor (that's a lot easier to use than vi). Add the following lines:

```
.mode column
.headers on
```

Then type `ctrl+x` this will ask if you want to save. Type: `y` followed by `return`.

Now whenever you type `sqlite3` in the command line any commands in this file will automatically execute.

## Using SQLite3 in Node

We need to install a database driver so we can interact with sqlite3 from NodeJS. To do this we'll install a module from npm. `cd` into your project directory and execute:

```
npm install --save sqlite3
```

You can read the API documentation for the sqlite3 module here: https://github.com/mapbox/node-sqlite3/wiki/API.

Next we need to create a directory to store our database so we can keep our project organized.

```
mkdir Database
```

Finally we need to add this directory to our `.gitignore` file since we don't actually want to push our entire database to our git repo. Open the `.gitignore` file and add the following line:

```
Database/
```

## Models

```
mkdir Models
```

This folder will contain our Models which we will use to interface with the database. All data should go through this interface. 

We will be using the **data access object** design pattern to provide an single interface to execute queries on our database. This way we have a consistent interface that all our models can use. You can read a bit on this pattern here https://www.tutorialspoint.com/design_pattern/data_access_object_pattern.htm (However just focus on the major ideas). Essentially it just acts as a layer between our code and the database. We are really only using a DAO so that we can **promisify** the methods that the sqlite3 module gives us. These methods are written using older style callbacks and wrapping them to use promises will significantly simplify development.

**If you haven't read up on promises you should check the MDN web docs here: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise. And check the notes on promises.**

### Data access object

```javascript
// This is in Models/dao.js
const sqlite3 = require('sqlite3')
const util    = require('util');

function createDB (dbFilePath) {
    return new Promise ( (resolve, reject) => {
        const db = new sqlite3.Database(dbFilePath, (err) => {
            if (err) {
                console.error(`Could not open ${dbFilePath}`);
                // Reject the promise if there is an error
                reject(err);
            }
        })
        // resolve the promise (i.e. keep the promise) if the db is 
        // created without error
        resolve(db);
    });
}


async function createDAO (dbFilePath) {
    const db = await createDB(dbFilePath);
    db.run = util.promisify(db.run);
    db.all = util.promisify(db.all);
    return db;
}

module.exports = createDAO;
```

I changed things around considerable since class by using NodeJS' built in utility for **promisifying** function (rather than wrapping it ourselves). However, we do still have to wrap the DB creation in a promise ourself. To **promisify** a function you just create a new `Promise` object which takes a function as its paremeter (which is the code being wrapped in a promise). This function takes 2 parameters `resolve` and `reject`. Both of which are functions.

You call one of these 2 within the function you are passing to the `Promise`. 

If there are any errors in the function, then you call `reject()` which will trigger the `.catch()` clause (or throw an exception if we use `async`/`await`). 

If the function should end normally instead of using `return` as you typically would, you instead call `resolve()` and pass it whatever you wanted to return. This will trigger the `.then()` clause (or the next line of code if we use `async`/`await`).

### Todont Model (primary table)

```javascript
// Models/Todont.js
class TodontModel {
    constructor (DAO) {
        this.DAO = DAO
    }
  
    createTable () {
        const sql = `
            CREATE TABLE IF NOT EXISTS todonts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            text TEXT,
            priority TEXT
        )`
        return this.DAO.run(sql)
    }

    add (todont, priority) {
        return this.DAO.run(
            'INSERT INTO todonts (text, priority) VALUES (?, ?)',
            [todont, priority]
        );
    }
    
    getAll () {
        return this.DAO.all(
            'SELECT text, priority FROM todonts'
        );
    }
}
  
module.exports = TodontModel;
```

This class wraps the SQL code we're using to store our todont objects. The constructor just takes a `DAO` object so that it can interface with the database. We start with 3 methods:

#### createTable ()
This method is used to create our database table. It has 3 columns:
1. id
    - each todont object has a unique id that automatically increments when we add a new row
2. text
    - this is the text of the todont that the user entered
3. priority
    - this is the priority level of the todont

Let's breakdown this create table query.

```sql
CREATE TABLE IF NOT EXISTS todonts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    text TEXT,
    priority TEXT
)
```

The first line is straightforward. It creates the table named `todonts` if it does not already exist. You should add `IF NOT EXISTS` whenever necessary because SQL with throw an exception if the table already exists.

The columns in the table are defined inside of the parenthesis of the create statement. The syntax for defining a column is:

`column_name DATE_TYPE [CONSTRAINTS]`

See the links below for info on data types and constriants. The constraits are in brackets here only to indicate that they are optional (you should not include brackets in your sql code).

```id INTEGER PRIMARY KEY AUTOINCREMENT,```
This defines a column named `id` that is an `INTEGER` type. It is the primary key in the table which means that it must be a unique value across all rows in this table. To insure the id is unique we use the `AUTOINCREMENT` constraint. This will automatically increment the id whenever we add a new row meaning we never have to explicitly set the id. Whenever we `INSERT` into the table the id is generated for us.

```sql
text TEXT,
priority TEXT
```

These 2 lines define 2 new rows named `text` and `priority` both of which are the `TEXT` datatype. 

That's all we need to create this table. 

#### add ()
```js
add (todont, priority) {
    return this.DAO.run(
        'INSERT INTO todonts (text, priority) VALUES (?, ?)',
        [todont, priority]
    );
}
```

The `add()` method is used to add data into the `todonts` table. We use the sql `INSERT` command to add data to tables. We just pass the sql query and data we are inserting to the `run()` method we **promisified** earlier.

We do have some new syntax here:

```js
this.DAO.run(
    'INSERT INTO todonts (text, priority) VALUES (?, ?)',
    [todont, priority]
);
```

The `run()` method can use **parameterized queries**. 

The typical syntax for an `INSERT` statement looks like this:

```sql
INSERT INTO todonts (text, priority) VALUES ('Don''t poke a sleeping bear.', 'low');
-- 'Don''t' <- The single quotes are used for strings and as the escape char in sql.
-- So the string above is stored as: Don't
```

So why bother with the question marks and parameters instead of using string concaenation or interpolation? Because what happens if one of the variables had malicious input? For example this:

```sql
this.DAO.run(
    `INSERT INTO todonts (text, priority) VALUES (${todont}, ${priority})`
);
```

```sql
this.DAO.run(
    `INSERT INTO todonts (text, priority) VALUES (${todont}, ${priority})`
);
```
what if todont was the following string:

```js
todont = "; Drop table todonts; --";
priority = "normal"
```

Then the results would be that the semicolon terminates the query you wanted to run. Then my code runs and a drop your `todonts` table then the `--` is sql syntax to indicate a comment (thus commenting out the rest of your query).

So we can't trust user input (we should **never** trust the user's input). So we need to sanitize all input before it hits our servers. However, this is cumbersome and a game of cat and mouse. Instead we can use paramterized queries which effectively eliminate SQL injection attacks. Always use parameterized queries like we're using on this project.

#### getAll()
```js
getAll () {
    return this.DAO.all(
        'SELECT text, priority FROM todonts'
    );
}
```

These queries just selects the `text` and `priority` for each row in the `todonts` table.


#### Links

For more information on the SQL commands and the categories they fall under:
https://www.geeksforgeeks.org/sql-ddl-dql-dml-dcl-tcl-commands/

There are 5 main commands that we will use most frequently.
```sql
CREATE -- used to create tables
SELECT -- used to retrieve data from tables
INSERT -- used to add data to tables
UPDATE -- used to update the data in a table
DELETE -- used to delete data from a table
```

Link to the sqlite documentation on supported datatypes: https://www.sqlite.org/datatype3.html#boolean_datatype.

The sqlite3 documentation can be pretty technical and the syntax diagrams are intimidating. However, it is worth it to spend some time to understand how to read a syntax diagram.
https://www.cs.nmsu.edu/~rth/cs/cs471/Syntax%20Module/diagrams.html

You may want to take a look at: https://www.sqlitetutorial.net/. This is a great site for learning SQL basics. Although I'll provide examples for the code we need, you can see more here.

## Using our models

We add the following lines to the index.js 

```js
const createDAO   = require('./Models/dao');
const TodontModel = require('./Models/TodontModel');


const dbFilePath = process.env.DB_FILE_PATH || path.join(__dirname, 'Database', 'Todont.db');
let Todont = undefined;
```

We can `require()` these files since we we used `module.exports` to give ourselves access outside the file.

At the end of our code I added 

```js
async function initDB () {
    const dao = await createDAO(dbFilePath);
    Todont = new TodontModel(dao);
    await Todont.createTable();
}
```

This async function lets us `await` on a promise. If you have a promise **and** you marked your function `async`. You can `await` on that code. It's like the `.then()` clause but more concise. It has the added benefit of making our code look more normal again.

Here I used it to wait on the DAO object to be created before we added the `TodontModel`. Finally we wait for the function to end.

I've made some changes and now the *listenting function* by making it async so the DB loaing can be waited on. Now we can wait for the DB to load before we start taking requests.

```js
// Listen on port 80 (Default HTTP port)
app.listen(80, async () => {
    // wait until the db is initialized and all models are initialized
    await initDB();
    // Then log that the we're listening on port 80
    console.log("Listening on port 80.");
});
```

## Adding client data to db:

Now we just passed the text and priority to the TodontModel's `add()` method. If the promise resolves we send back status `200` (OK) othwerise we send status `500` (internal server error)

```js
app.post("/add_todont", (req, res) => {
    const data = req.body;
    console.log(data);
    
    Todont.add(data.text, data.priority)
        .then( () => {
            res.sendStatus(200);
        }).catch( err => {
            console.error(err);
            res.sendStatus(500);
        });
});
```

Remember we wrapped `add()` to return a promise so we can use the `.then()` clause to send back `200` or the `.catch()` clause for `500`. We can also use `async`/`await` instead of `.then()` like this:

```js
//                          v----- Mark this function async so you can use await
app.post("/add_todont", async (req, res) => {
    const data = req.body;
    console.log(data);
    try {
        await Todont.add(data.text, data.priority); // wait for add() to complete
        res.sendStatus(200);                        // send back success status
    } catch (err) {                                 // if there's an error the catch it
        console.error(err);
        res.sendStatus(500);                        // and send status 500
    }
});
```

We have to use a standard try...catch with `async`/`await` since there's no `.catch()` block.

### getAll()

We also modify the `GET /todont_items` route so that it uses the database.

```js
app.get("/todont_items", (req, res) => {
    Todont.getAll()
        .then( (rows) => {
            console.log(rows);
            // remember to change index.js
            res.send(JSON.stringify({todont_items: rows}));
        })
        .catch( err => {
            console.error(err);
            res.sendStatus(500);
        })
});
```

Now we just use the `getAll()` method so we can send them back to the client. It's the largely the same except now we are using promises to get the info from the database.

We don't need to make **any** changes on the client side. We just ensure that we send back the data in the same *shape* that the client is expecting and everything just works. This is a massive advantage get by using MVC and adhering to seperation of concerns.

