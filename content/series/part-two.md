---
title: "Create a User Model"
date: 2022-03-01T21:43:00-06:00
draft: false
series: ""
step: 2
summary: "Create a user model, add fields and methods, and test"
---

In this lesson, we will use the Buffalo CLI to generate a User model with custom some fields, write some methods for User, and begin writing tests with the Buffalo test suite.

## Prerequisites

* install Buffalo according to the official site instructions
* have access to one of the supported database versions (CockroachDB, MySQL, PostgreSQL, SQLite). This could be installed on your local development machine or using a cloud solution that is accessible.
* have some familiarity with entering commands in the terminal
* install a code editor to modify files

## Resources

<a href="https://www.youtube.com/watch?v=SwZoj4KEnvE&list=PL7fZGRmlHt5ldUTseGiwpG_-IjA7Yv143&index=2&t=327s">Watch the video walkthrough</a>

<a href="https://github.com/briwagner/learn-buffalo/tree/part-2">View the code</a>

## Step 1: Create a project

Enter `buffalo new PROJECT_NAME` to generate the skeleton for a new project. By default, Buffalo assumes you are using PostgreSQL, and the generated project files will reflect that. If you are not, then add `--db-type mysql` to the command above to specify MySQL or another DB platform.

Use `buffalo new --help` for more instructions.

Buffalo uses Go modules to manage dependencies, so one of the first files to notice is `go.mod` in the root of the project. If you see any packages that are missing, try typing `go mod tidy` in your terminal to resolve those dependencies.

Next, make any necessary changes or overrides in these two files:

* `database.yml`: holds database credentials
* `.env`: add other environment settings, like custom HTTP port settings, or global variables

Note the <a href="https://github.com/gobuffalo/pop">Buffalo Pop</a> plugin, which manages the database work, can take the database information as separate pieces (user, password, database name, etc.) or as a fully-formed connection string. The `database.yml` file that is generated for your database platform should have examples of both.

## Step 2: Create the database

Using a UI tool or the CLI for your specific database, we need to create a user and password for this project. We can also create the databases at this point &mdash; one for the dev environment, and another for test. Or we can do it later with the Buffalo CLI.

Note: the <a href="https://github.com/briwagner/learn-buffalo/tree/part-2"> demo project repo</a> uses a MariaDB database.

For MariaDB, we can execute these commands in the database CLI:

`CREATE DATABASE cool_project;` (optional)

`CREATE USER buffalo@localhost IDENTIFIED BY 'special_password';`

`GRANT ALL PRIVILEGES ON cool_project.* TO buffalo@localhost;`

`GRANT ALL PRIVILEGES ON cool_project_test.* TO buffalo@localhost;`

Now we need to enter this information in the `database.yml` file above.

After that, we can run `buffalo pop create` to generate the dev database if you didn't do that already. If it already exists, you will see an error message with that information. If it fails for another reason, make sure that your DB user has the proper rights to create the database specified.

## Step 3: Create a User Model

Let's start writing some code for our project!

Before we have page templates or routes to worry about, we need to create some data models for our site. This data is stored in the database, and the Buffalo Pop plugin will help us create, update, read, and delete that data for the site. If you remember the MVC discussion from Part One, this is the Model part of that paradigm.

In the terminal, use this command to generate a User model.

`buffalo pop g model user`

What is happening here? We execute the `pop generate` command. Notice how "g" is short for "generate"; we can use both interchangeably. What do we want to generate? A model called "user".

Our project code should now have some new files in it.

* models/user.go
* models/user_test.go
* migrations/{date_stamp}_create_users.up.fizz
* migrations/{date_stamp}_create_users.down.fizz

The first file has our User struct, and it's where we will add our application logic later. The second is for writing some tests against that logic. We also have two migration files &mdash; one for "up" to add the User table and columns, and another for reversing that action. See Step 4 below.

Open the `models/user.go` file and see what the User struct looks like.

<pre class="code-block">
// User is used by pop to map your users database table to your go code.
type User struct {
	ID        uuid.UUID `json:"id" db:"id"`
	CreatedAt time.Time `json:"created_at" db:"created_at"`
	UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
</pre>

This is the default set of fields that Buffalo will create for any new model. You can add fields here, but remember to modify the migration file so that those columns are generated in the database as well. If you want fields that don't add or read from the database then add this tag: `db:"-"`.

Another option for adding fields is to do that when we call `buffalo pop g model user`. We can append to this command and specify additional fields to create. By default these fields are type `string` but we can override that:

`buffalo pop g model user first_name last_name age:int`

Now our model looks like this:

<pre class="code-block">
// User is used by pop to map your users database table to your go code.
type User struct {
	ID        uuid.UUID `json:"id" db:"id"`
	FirstName string    `json:"first_name" db:"first_name"`
	LastName  string    `json:"last_name" db:"last_name"`
	Age       int       `json:"age" db:"age"`
	CreatedAt time.Time `json:"created_at" db:"created_at"`
	UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
</pre>

And the migration "up" file has this:

<pre class="code-block">
create_table("users") {
	t.Column("id", "uuid", {primary: true})
	t.Column("first_name", "string", {})
	t.Column("last_name", "string", {})
	t.Column("age", "integer", {})
	t.Timestamps()
}
</pre>

## Step 4: Run the Database Migration

Let's put this structure into the database!

Run `buffalo pop migrate up`. We should see confirmation that the migration was run. And logging in to the database cli, we should see that table was created with the fields we specified.

With MariaDB, for example, type: `show tables` and `describe users`.

Another handy command we can use during development is `buffalo pop reset`. This will drop the existing tables, and run the migration files in their proper order. <strong>Note you will lose any data stored in the database!</strong> But this is helpful if you return to a model later, and decide to make some changes, like adding or renaming a field.

## Step 5: Add Logic to the User Model

Just to get us rolling as we build out our User model, we can add a short method to return the full name of a user.

<pre class="code-block">
func (u User) FullName() string {
	return fmt.Sprintf("%s %s", u.FirstName, u.LastName)
}
</pre>

This is something we can test! But first we need to do a little cleanup:

First, double-check your package versions for Buffalo "suite" and "validate". They should be running the latest version (suite v4 and validate v3 as of mid-2022) and sometimes the buffalo generator defaults to an old version. You can edit these imports in the relevant files, and run `go mod tidy` to resolve the dependencies. <a href="https://github.com/gobuffalo/suite">Suite</a> and <a href="https://github.com/gobuffalo/validate">Validate</a>

Second, remove the default test in the `models/user_test.go` file. Buffalo generates tests that are always doomed to fail as a reminder so that we write some real tests for our application logic. Remove this line:

<pre class="code-block">
ms.Fail("This test needs to be implemented!")
</pre>

We will never get that one to pass!

## Step 6: Write Tests for the User Model

The Buffalo package includes an entire test suite with each project. Execute `buffalo test` to run all the tests available.

What is happening? Buffalo uses the "test" credentials in the `database.yml` file and tries to connect to the database. (Note that "test" is not a separate environment or tier, as might exist in other situations. Rather it's the resource to use when running the test suite.) Once connected, the Buffalo suite drops the tables (if they exist) and rebuilds them from the schema file before running each test.

Another option, instead of running all the tests, is to specify a package to test:

`buffalo test ./models`

Or name a single function to test:

`buffalo test -m "Test_User"`

Now, let's write a test for that User method we added above. The name of our test functions can be anything &mdash; as long as it starts with "Test_". But it's good practice to provide names that describe the behavior you are testing.

In `models/user_test.go`, add the following:

<pre class="code-block">
func (ms *ModelSuite) Test_User() {
	u := &User{
		FirstName: "Nikola",
		LastName:  "Tesla",
	}

	ms.Equal("Nikola Tesla", u.FullName(), "FullName returns user name.")
}
</pre>

If we run `buffalo test ./models`, we should see this test pass. Hooray!

Writing our code in this order doesn't follow the advice of the <a href="https://en.wikipedia.org/wiki/Test-driven_development">test-driven development pattern</a> if that's important to you. Instead we expect to write a test first, run it to confirm it fails, and finally write the logic to make the test pass.

In that case, you can modify `FullName()` to return an empty string. Now run the test and it should fail. Restore the original code, and confirm it passes.

Going further, we can write tests that check our integration with the database. To do that, we'll need a database connection that the Buffalo suite provides via the `ModelSuite`, passed into each test function:

<pre class="code-block">
func (ms *ModelSuite) Test_User() {
  // Get the database connection.
  db := ms.DB
  // Run tests.
  ...
}
</pre>

Now continue building out our test function, so the file looks like this:

<pre class="code-block">
package models

func (ms *ModelSuite) Test_User() {
	u := &User{
		FirstName: "Nikola",
		LastName:  "Tesla",
	}

	ms.Equal("Nikola Tesla", u.FullName(), "FullName returns user name.")

	db := ms.DB
	err := db.Create(u)
	if err != nil {
		panic(err)
	}

	ms.NotNil(u.ID, "User ID is generated when saved to DB.")
}
</pre>

With this addition, we are saving the User to the database and checking any error. So what are we testing here? Note that we don't set the ID when we set the first and last name values. By checking the ID on the final line above, we confirm that it's being assigned during the call to `db.Create(u)`.

This is a very simple test pattern. We can do better.

## Step 7: Model Validators

In many cases, we want to avoid empty values when saving data to the database. Each of our Users should always have a first and last name and an age, for example. How do we enforce that?

Inside the `models/user.go` file, we can see that Buffalo has helped us get started. There are skeletons for three methods to validate data <em>before</em> it is saved to the database. We can add logic inside the `Validate` method which will run for all operations to create or update the data. If we have additional checks that should run <em>only</em> during a create operation, then we add that to `ValidateAndCreate`. Similar for `ValidateAndUpdate` for updates.

Notice that each of these methods return a function and an error. So when adding custom validations we want to follow that pattern.

<pre class="code-block">
// Validate gets run every time you call a "pop.Validate*" (pop.ValidateAndSave, pop.ValidateAndCreate, pop.ValidateAndUpdate) method.
func (u *User) Validate(tx *pop.Connection) (*validate.Errors, error) {
	var err error
	return validate.Validate(
		&validators.StringIsPresent{Field: u.FirstName, Name: "FirstName"},
		&validators.StringIsPresent{Field: u.LastName, Name: "LastName"},
		&validators.IntIsPresent{Field: u.Age, Name: "Age"},
	), err
}
</pre>

The <a href="https://github.com/gobuffalo/validate">Buffalo validate</a> package has several other built-in validators, as well as custom functions we can write. Check <a href="https://pkg.go.dev/github.com/gobuffalo/validate/v3/validators">the docs</a> for more information.

Now, if we run our test again, we won't see any change. We aren't setting an age in the test case, so the validation `IntIsPresent` should trigger an error. What happened?

Our original code uses the `db.Create()` method, which doesn't run any of the model's validators. This can be handy during imports, when we need to insert data without running checks &mdash; hopefully you verify that data some other way! But for writing tests, we often want to reach for the pop.`ValidateAndCreate()` method. Let's fix that:

<pre class="code-block">
package models

func (ms *ModelSuite) Test_User() {
	u := &User{
		FirstName: "Nikola",
		LastName:  "Tesla",
		Age: 42,
	}

	ms.Equal("Nikola Tesla", u.FullName(), "FullName returns user name.")

	db := ms.DB
	verrs, err := db.ValidateAndCreate(u)
	if err != nil {
		panic(err)
	}

	ms.NotNil(u.ID, "User ID is generated when saved to DB.")
	ms.False(verrs.HasAny(), "User cannot be created without age field.")
}
</pre>

See how `ValidateAndCreate` has two return values? The first is the validation errors from the validator, which are helpful when we are processing an HTML form and we want to return them to the user to properly complete the form. We'll see how those work in <a href="/series/part-six">Part Six: Forms</a>.

In our test case, we can update our User to include an age, and then verify that no `verrs` come back.

## Step 8: Going further

This is just the start of writing tests for our models. To learn more, check the documentation:

* <a href="https://pkg.go.dev/github.com/gobuffalo/suite">Buffalo suite</a>
* <a href="https://pkg.go.dev/github.com/gobuffalo/validate">Buffalo validate</a>

<a href="https://www.youtube.com/watch?v=SwZoj4KEnvE&list=PL7fZGRmlHt5ldUTseGiwpG_-IjA7Yv143&index=2&t=327s">Watch the video</a>

<a href="https://github.com/briwagner/learn-buffalo/tree/part-2">View the code</a>