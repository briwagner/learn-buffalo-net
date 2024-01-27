---
title: "Integrate VueJS"
date: 2024-01-03T18:27:04-06:00
draft: false
tags: []
summary: "Add VueJS components to your Buffalo site for interactive and dynamic pages"
---

It's very common to see JavaScript tools like <a href="https://vuejs.org">VueJS</a> or <a href="https://react.dev">React</a> in use on the frontend of many websites today. JS components can help bring dynamic and interactive experiences for users. When it comes to integrating those tools with Buffalo, what does that look like?

This recipe looks at two ways to integrate VueJS components with a Buffalo site. Both involve using Buffalo to generate HTML pages that embed Vue components.

## What Are We Building: Event Planner app

The Event Planner app allows registered users to create events with a title, description and date. Anonymous users can register for any event, using their email. The event detail page shows a list of guests who have registered.

So, using Buffalo tools, we create a site with the following pieces:

* events page with a static list of items
* event detail page with a plain HTML form, for guest registration
* guest registration form sends a POST request and reloads the page with the newly rebuilt list of guests

From our starting point above, we will add some VueJS components.

1. The event list page would be more useful with a search-by-title function
1. Validation for the guest-registration is quicker if it happens in the browser without hitting the server
1. After submitting the form, our changes load faster if we don't reload the entire page, showing a success/error message instead

<div class="ui segment secondary">
<h3>Resources</h3>

<p>Project code is here: <a href="https://github.com/briwagner/buffalo-vue-integration">Buffalo Vue Integration</a></p>

<p>This was built with Buffalo version 18.14. It was also built with a <a href="https://github.com/briwagner/buffalo-auth">version of the Buffalo Auth plugin</a>.</p>

<p><strong>Note</strong>: this app was built with Webpack disabled, just to simplify the examples. The main Buffalo template `application.plush.html` includes a script tag to download VueJS and Axios from a CDN. There is benefit to using Webpack's bundling in a production app, but it would change some of the setup used here.</p>
</div>

<div class="ui segment red">
<h3>Multiple Servers, Decoupled Frontend and Backend, etc.</h3>
<p>On large-scale projects, it's very common to see fully decoupled sites, where a JavaScript-driven frontend is delivered on one server, and the backend data is served on another. In this case, the development and deployment of each stack is often independent of the other. A framework like Buffalo doesn't have any magic to make that easier. Those types of decoupled projects involve special server configuration, in part to satisfy CORS policies. Those topics go beyond learning Buffalo, so we aren't discussing that here.</p>
</div>


## Buffalo Serves Everything

*Who is this for?*

Let's imagine you're a solo developer and you need to get this project working quickly. You don't want to build a decoupled site and manage two servers and two build systems. Or you don't want to set up special server configuration needed for two-lane traffic to your site.

You decide to build one Buffalo app to host everything. Let's go!

### Vue component uses JSON data on the page

For many websites with JavaScript frontends, JSON data is the foundation for these components. Often, that data comes from HTTP requests to dedicated endpoints that serve only JSON data. But ... it doesn't have to be that way. Especially for data that doesn't change so frequently, loading it along with the rest of the HTML page data works just fine.

Take our list page of events. Right now it shows a short version of each event -- including the title and date -- with a link to the detail page. If we convert it to a VueJS component, we can make it interactive by adding a filter box.

The controller for the list page queries the database for the events and passes that to the template:

File: `actions/events.go`

{{< codeblock >}}
// EventsListHandler returns GET for list of all events.
func EventsListHandler(c buffalo.Context) error {
	tx := c.Value("tx").(*pop.Connection)
	events := models.Events{}

	err := tx.All(&events)
	if err != nil {
		log.Print(err)
		return c.Redirect(301, "/")
	}

	c.Set("events", events)
	return c.Render(http.StatusOK, r.HTML("events/all"))
}{{< /codeblock >}}

So instead of passing the slice of Event structs, we can marshal to JSON and pass that string to the page template. Now instead of making a separate HTTP call for that data, it's loaded on the page, where the user can't see it but our Vue component can.

{{< codeblock >}}
// EventsListHandler returns GET for list of all events.
func EventsListHandler(c buffalo.Context) error {
	tx := c.Value("tx").(*pop.Connection)
	events := models.Events{}

	err := tx.All(&events)
	if err != nil {
		log.Print(err)
		return c.Redirect(301, "/")
	}

	// Marshal to JSON so the Vue app can read it.
	data, err := json.Marshal(events)
	if err != nil {
		log.Print(err)
		return c.Redirect(301, "/")
	}
	c.Set("events", string(data))
	return c.Render(http.StatusOK, r.HTML("events/all"))
}{{< /codeblock >}}

What does that look like in the template? A couple things to keep in mind:

* wrap the JSON string in a script tag
* use the Buffalo helper `toJSON`, so it properly escapes the quotation marks
* make sure the JSON string is loaded on the page *before* the component (eventList.js) that relies on this data

`templates/events/all.plush.html`

{{< codeblock >}}
<script>
  let eventList = <%= toJSON(events) %>
</script>

<%= javascriptTag("eventList.js") %>{{< /codeblock >}}

Now let's shift to our Vue component. First, the component's `mounted()` stage will look for a variable called `eventList`. Then call `JSON.parse()` to turn a string back into a JSON object, and finally output an array of event objects that Vue can manipulate.

`public/assets/eventList.js`

{{< codeblock >}}
  ...
  data() {
    return {
      events: [],
    }
  },
  ...
  mounted() {
    let events = [];
    const data = JSON.parse(eventList)
    if (!Array.isArray(data)) {
      console.log("not an array")
    }
    else {
      for (let i = 0; i < data.length; i++) {
        let d = new Date(data[i].Date).toLocaleDateString('en-us', {
          year: 'numeric',
          month: 'short',
          day: 'numeric',
          hour: '2-digit',
          minute: '2-digit'
        })
        const item = {
          Title: data[i].Title,
          Link: "/events/" + data[i].id,
          EventDate: d
        }
        events.push(item)
      }

      this.events = events;
    }
  },
{{< /codeblock >}}

The <a href="https://github.com/briwagner/buffalo-vue-integration/blob/master/public/assets/eventList.js">full example is here</a>, including the template and the filtering method. Our focus on this site is Buffalo, so we won't spend much time explaining VueJS.

**What have we done?**

 On a single page request, we have loaded our data payload along with a JavaScript component, and some HTML elements. No extra requests are needed, and this page is self-contained in a way that allows it to be stored in a page-cache, if we have one.

### Vue component fetches JSON from a backend route

Let's modify our Vue components to fetch data from an HTTP endpoint instead of reading it from the server-rendered page. Our first step is to add the ability to make HTTP requests, and for that we'll add the <a href="https://axios-http.com/docs/intro">Axios library</a> to our project. One simple way is to link to a CDN in our page template `templates/application.plush.html`. Another option is to use <a href="https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API">Fetch API</a>.

Where is that data going to come from now? We'll need to add routes to our Buffalo app that serve only the JSON data we need.

`actions/events.go`

{{< codeblock >}}
// EventsListHandler returns JSON list of all events.
func EventsListJSONHandler(c buffalo.Context) error {
	tx := c.Value("tx").(*pop.Connection)
	events := models.Events{}

	err := tx.All(&events)
	if err != nil {
		log.Print(err)
		return c.Redirect(301, "/")
	}

	// r.JSON handles Marshal for us.
	return c.Render(http.StatusOK, r.JSON(events))
}
{{< /codeblock >}}

Add a line in `actions/app.go` to serve this handler on the path for `events/json`. And this is all we need to send JSON data from Buffalo. But in our case, we already have a route that returns a list of Events, even though it's in HTML. Can we re-use it? Yes!

Buffalo controllers allows us to respond to different kinds of requests, by checking the "content-type" on the request. So instead of adding a new route, we could add some logic in our existing controller so that it satisfies both needs at the `/events` path.

`actions/events.go`

{{< codeblock >}}
// EventsListHandler returns GET for list of all events.
func EventsListHandler(c buffalo.Context) error {
	tx := c.Value("tx").(*pop.Connection)
	events := models.Events{}

	err := tx.All(&events)
	if err != nil {
		log.Print(err)
		return c.Redirect(301, "/")
	}

  // Check request type.
	ct, _ := c.Value("contentType").(string)
	if ct == "application/json" {
		return c.Render(http.StatusOK, r.JSON(events))
	}
  ...
  // Continue to render HTML as before.
}
{{< /codeblock >}}

Ok, so the Buffalo route is ready. Now let's modify our VueJS component to fetch that data via HTTP request, and not expect to find it on the page. We'll create a new version of the component which is served at the `/events-remote` path of our app. Much of the code is the same, except where we fetch data and parse it into Event objects.

`public/assets/eventListRemote.js`

{{< codeblock >}}
  methods: {
    async getEvents() {
      const resp = fetch('/events/', {headers: {'Content-Type': 'application/json'}});
      return (await resp).json();
    }
  },
  async mounted() {
    let events = [];
    let data = [];
    // Make request to load event list.
    await this.getEvents().then(res => data = res);
    ...
  }
{{< /codeblock >}}

Notice how we have to use `async` on the `mounted()` method so the request is completed with the corresponding `await`.

This example uses Fetch API, but Axios would work too. Also notice we are hitting the `/events` route which is smart enough to know we want JSON back and not an HTML page. But we could also have our component fetch that data from `/events/json`.

### VueJS Form

Let's combine the ideas above to build a <a href="https://github.com/briwagner/buffalo-vue-integration/blob/master/public/assets/eventForm.js">form to register guests</a> which is available at the `/app` route. We can use the server to render JSON data on the page, which will build a select list. And with the help of Axios, our Vue component will make a POST request to a custom endpoint. Compared to a server-generated HTML form, there are a couple benefits to using a Vue component for forms:

* greater ability for form validation and user feedback
* eliminate reloading the entire page to display success/error of the request

**Note**: one critical piece for submitting forms with JavaScript components is the `authenticity_token`. By default, Buffalo expects all forms to have that token set with the <a href="https://github.com/gobuffalo/mw-csrf">CSRF package</a> -- something that Buffalo forms does for you. You can disable that in Buffalo. Or you can make sure the Vue component grabs the token from the page and includes it in the POST request back to Buffalo. Notice how we do that in the <a href="https://github.com/briwagner/buffalo-vue-integration/blob/master/public/assets/eventForm.js">eventForm.js</a>.

`public/assets/eventForm.js`

{{< codeblock >}}
...
  data() {
    return {
      form: {
        EventID: '',
        FullName: '',
        Email: '',
        authenticity_token: ''
      },
    }
    ...
  }
{{< /codeblock >}}

We shadow the elements of the Buffalo form in our `data()` component state, *including* the `authenticity_token`. And in the `mounted()` stage, we grab it from the server-rendered page.

{{< codeblock >}}
let tok = document.querySelector('meta[name="csrf-token"]').content;
this.form.authenticity_token = tok;
{{< /codeblock >}}

In this way, Buffalo gets the token it expects and our request is handled successfully.

The final piece of this implementation is a custom route that handles the VueJS form structure. This is not necessary, though! If you already have a route that creates a server-side form, and another one that handles the form submission, then you can re-use it. The only requirement is that the structure of your VueJS component should match that of the server-side form.

For this example, the Vue app submits only the required data to create a guest registration. To handle that simplified structure, we have an AppForm struct in the handler that makes it easy to parse the form data.

`actions/events.go`

{{< codeblock >}}
type AppForm struct {
	EventID  uuid.UUID `form:"EventID"`
	FullName string    `form:"FullName"`
	Email    string    `form:"Email"`
}

// AppFormHandler responds to POST to add-guest for Vue form.
func AppFormHandler(c buffalo.Context) error {
	tx := c.Value("tx").(*pop.Connection)
	req := &AppForm{}
	err := c.Bind(req)
	if err != nil {
		log.Printf("form bind error %s", err)
		return c.Render(400, r.String("form binding error"))
	}

	event := &models.Event{}
	err = tx.Find(event, req.EventID)
	if err != nil {
		log.Printf("error finding event")
		return c.Render(404, r.String("event not found "+req.EventID.String()))
	}
  ...
}
{{< /codeblock >}}

Notice in <a href="https://github.com/briwagner/buffalo-vue-integration/blob/master/actions/events.go">events.go</a> that `AppFormHandler` is the handler for the Vue form submission. It's very similar to the logic and flow of `EventAddGuestHandler` which handles the server-rendered form. We could go one step farther and move some of that logic into the Events model, or find another method in the actions package to avoid the duplicated code. Work for another day!

## Going Further

<div class="ui segment secondary">
<p>This example project is fairly simple. What would make it even more useful?</p>

<ul>
<li>connect to an email delivery system to send password recovery messages, account confirmation, event reminders, etc.</li>
<li>add a reminder system</li>
<li>allow editors to set a limit on the number of reservations</li>
<li>add a pending/accepted status on Guests, to allow admins to control attendance</li>
<li>allow guests to login and cancel their registration</li>
</ul>

<p>Feel free to take this simple project and make it better!</p>
</div>