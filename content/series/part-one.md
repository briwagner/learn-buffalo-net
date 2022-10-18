---
title: "Create a New Project"
date: 2022-03-01T21:31:17-06:00
draft: false
series: ""
step: 1
summary: "Create a new project, add static content, use dynamic route parameters."
---

This lesson will introduce you to the Buffalo framework, create a new project, add some routes and templates, and generate a binary for the application.

## Prerequisites

* install Buffalo according to the official site instructions
* have some familiarity with entering commands in the terminal
* install a code editor to modify files

## Resources

<p><a href="https://www.youtube.com/watch?v=1mXWtP3EkLk&list=PL7fZGRmlHt5ldUTseGiwpG_-IjA7Yv143&index=1" class="link-border">Watch the video walkthrough</a></p>

<p><a href="https://github.com/briwagner/learn-buffalo/tree/part-1" class="link-border">View the code</a></p>

## What is Buffalo?

Buffalo is an open-source framework written in Golang. It's not one single package, it's really a whole suite of tools that can be used individually or in combination to build web applications. Some of these packages were created by the Buffalo community and others are managed by others. All together, a Buffalo project allows us to manage the following components:

* database ORM (object relational mapping) with Buffalo Pop
* HTML templates with Plush
* HTML forms
* custom routing
* web browser sessions and cookies
* front-end toolchain with Webpack, Bootstrap and JQuery
* background tasks
* testing suite

Install Buffalo following the instructions on the official site. Once that's done, we can test some commands in a shell.

`buffalo version` will display the version number.

`buffalo` with no other argument shows a list of available commands.

## Step 1: Create a project

Enter `buffalo new PROJECT_NAME` to generate the skeleton for a new project and begin exploring the files created.

Buffalo uses Go modules to manage dependencies, so one of the first files to notice is `go.mod` in the root of the project. If you see any packages that are missing, try typing `go mod tidy` in your terminal to resolve those dependencies.

Your new project includes a README.md file in the root which includes a lot of handy instructions about how to get started building an app. This includes how to add custom settings or overrides in two files in your new project:

* `database.yml`: holds database credentials
* `.env`: add other environment settings, like custom HTTP port settings, or global variables

### Model, View, Controller

Buffalo shares the MVC pattern that is used by other frameworks like Ruby on Rails, Angular and others. This is a way to describe the three separate parts of the application, and help understand the different responsibilities they have.

**Model** refers to the data structures and, often, how they are stored in a database. For example, a User model may have fields like email, name and ID, which are stored in one or more database tables. The Model portion is contained in the `models/` directory, where you will see several files included in a new app.

**View** is the part of the application that end-users will experience, most likely in the form of HTML page content. The pieces related to View live in the `templates/` directory.

**Controller** sits in the middle, taking user requests and matching it to data managed by the application. This is where much of the application logic resides, helping us to manage dynamic data. And `actions/` is the folder that holds our Controllers, including the main logic file for any Buffalo app, the `app.go` file.

## Step 2: Start a Dev Server

From your terminal, enter `buffalo dev` to start a dev server that rebuilds the app as we work. The Buffalo engine will try to serve the app on the default port of 3000, which you can view in a web browser on your dev machine. If you have something else running on that port, you will see an error message in the terminal and the app will be unavailable. Open the `.env` file and add a line like `PORT=3022`, for example, to serve the app on another port.

If you followed the instructions in the README.md about creating a database and adding those credentials to your `database.yml` file, then you should see the start page in your browser. Hooray!

If not, you will see an error page. That's because Buffalo tries to maintain a database transaction for every page request it receives. Since Buffalo cannot connect to a database, it throws an error. To overcome this for now, look in the `actions/app.go` file for this line:

`app.Use(popmw.Transaction(models.DB))`

Delete it for now, or convert into a comment by adding two "/" characters.

`// app.Use(popmw.Transaction(models.DB))`

Now Buffalo will not create that database transaction on every page request.

## Step 3: Create a Custom Route

Look at that `actions/app.go` file again to see the default route: `app.GET("/", HomeHandler)`. This syntax means our app takes a "GET" request to the root location and passes it off to an HTTP handler function that is called `HomeHandler`. By default, that handler is defined in another file in the `actions/` directory. It doesn't really matter which file, just that it resides in the `actions` package.

Let's add a new route in the `app.go` file, similar to the one above. Define a path like `/about` and provide a named handler that matches well with the path, something like `AboutHandler`.

Next, define that handler in the `actions/home.go` file, where the original `HomeHandler` function resides. If you're familiar with HTTP handler functions in Go, then Buffalo handlers may look a bit different. That's because Buffalo has a special Context type that manages the initial request and parameters, as well as the response and other values.

<pre class="code-block">
func AboutHandler(c buffalo.Context) error {
	fmt.Println(c.Request().URL)    // => "/about/"
	fmt.Println(c.Request().Method) // => "GET"

	c.Set("title", "About page")
	c.Set("content", "This is the about page.")
	return c.Render(http.StatusOK, r.HTML("about.html"))
}
</pre>

See above how we can retrieve values from the context, and set values, like data retrieved from a database or an API request. These values are then available to the next HTTP handler, on in this case, passed to an HTML template. The Buffalo context also has methods! Just like with `HomeHandler`, we return the context, calling the `Render()` function, including the name of an HTML template file.

*So where is that template?*

Create a new file in templates: `about.plush.html`. Here we can write HTML, or use Plush syntax to print or process data from the context. The syntax is different from the Go standard library, but the basics are the same.

<pre class="code-block">
&lt;h1&gt;<%= title %>&lt;/h1&gt;
&lt;p&gt;<%= content %>&lt;/p&gt;
</pre>

## Step 4: Create a Dynamic Route

Let's take our routes one step further and make a *dynamic route*. What does that mean? It's when we define a placeholder in the URL that we expect to have any number of values, something like `/user/1`, `/user/2`, `/user/3`. The "/user" portion doesn't change but the ID does.

Let's create that route and add a handler. Now our `app.go` file looks like this:

<pre class="code-block">
    app.GET("/", HomeHandler)
    app.GET("/about", AboutHandler)
    app.GET("/user/{user_id}", UserHandler)
</pre>

The next step is to build out our handler where we will have some code to process the dynamic value `user_id`. For simplicity, we can add to the existing `actions/home.go` file, or feel free to create a file called `actions/user.go`.

<pre class="code-block">
type User struct {
	ID        int    `json":id"`
	Email     string `json:"email"`
	FirstName string `json:"first_name"`
	LastName  string `json:"last_name"`
}

func UserHandler(c buffalo.Context) error {
	user_id := c.Param("user_id")
	user, err := getUserInfo(user_id)
	if err != nil {
		c.Flash().Add("warning", "Bad connection")
		c.Redirect(301, "/")
	}
	c.Set("user", user)
	return c.Render(http.StatusOK, r.HTML("user.html"))
}
</pre>

There's a gotcha here. I have a mystery function, `getUserInfo` that fetches data for the user ID. It could be from a database or some third-party service, but let's not worry about that. Assume we get a `User` struct and an error back from it. A check of the error allows us to process a redirect, or we can set this new value on the context and render an HTML page.

Again, notice that Buffalo context is doing a lot of work here. We get the `user_id` from the context -- the variable name here should match the one we define in the route definition. And the redirect is actually a method called on the context.

File: `templates/user.plush.html`

<pre class="code-block">
&lt;h1&gt;<%= user.FirstName %> <%= user.LastName %>&lt;/h1&gt;
&lt;p&gt;<%= user.ID %>&lt;/p&gt;
&lt;p&gt;<%= user.Email %>&lt;/p&gt;
</pre>

More about Plush syntax: @todo add link

One final thing about HTML templates. The default app provides a partial template `templates/_flash.plush.html`, which is used to show user-specific messages. This is a pattern you may have seen with other web frameworks. We can add a flash message to a page response with the `c.Flash().Add()` method as we show above. The first parameter in that method is the message status (e.g. "warning", "notice"), an arbitrary string that can help control how messages are displayed. The default templates are designed to work with Bootstrap classes @todo: add list of class names. But you can create your own status levels and modify this template however you want.

Bootstrap flash message @todo add link

## Step 5. Build our App

The final step is to prepare our application for the real world. Since Go applications are compiled ahead of time, we need to do that here and we'll use the Buffalo CLI to do it. From the root of your project, type this in your terminal: `buffalo build`

This command will use the default settings ("development" environment) and generate a binary in the `bin/` directory. The binary contains all the logic, templates and front-end assets needed for our app to run. There are some helpful flags available to `build` as well:

* `--environment <string>`: target a different environment, e.g. test, production
* `--clean-assets`: reduce the binary's file size by deleting stale files generated previously by Webpack
* `-o <string>`: specify the name of the binary, e.g. myapp-v1.0, else it uses the project name

One thing that *does not happen* with the build command is adding the environment variables we may have used during development. If we need to add the same ENV vars or create new ones, we must create a new file called `.env` in the same directory as the binary. When the binary starts, it will look for that file and apply those variables to the application. (There are other ways to run the Go binary and pass ENV variables, which we discuss in later lessons.)

Now, let's see if the app works! From the project root, type `./bin/<appname>` and the app should start up. Or change directories into `bin/` and run the app: `cd bin && ./<appname>`

## Review

This walkthrough covered a lot of ground:

1. basics of MVC web applications
1. create a new Buffalo project
1. add static routes
1. add custom HTML templates
1. create dynamic routes
1. run a dev server
1. generate a production binary

This is just a start. In other lessons, we'll work on connecting to a database, creating models to handle data, writing tests, and making more HTML templates.

<p><a href="https://www.youtube.com/watch?v=1mXWtP3EkLk&list=PL7fZGRmlHt5ldUTseGiwpG_-IjA7Yv143&index=1" class="link-border">Watch the video</a></p>

<p><a href="https://github.com/briwagner/learn-buffalo/tree/part-1" class="link-border">View the code</a></p>