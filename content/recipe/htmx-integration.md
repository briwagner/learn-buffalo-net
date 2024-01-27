---
title: "JavaScript Doesn't Have to Be Hard: htmx Integration"
date: 2024-01-14T14:54:44-06:00
draft: false
tags: []
summary: "The htmx library is JavaScript made easy, providing out-of-the-box interactions with Buffalo or any web app"
---

JavaScript is a key component of any modern web application today. It delivers interactive experiences that are custom-fit for users, but often it comes with a high cost in developer hours. For solo devs or small teams, it can be a challenge to find the right balance between providing enough JS interaction to users, without investing a lot of time and effort.

The <a href="https://htmx.org/">htmx library</a> tries to find that middle ground, somewhere between a handful of custom scripts and a full framework like Angular, VueJS or React. In the words of the folks behind it:

<div class="ui raised segment blockquoted">
<p>htmx gives you access to AJAX, CSS Transitions, WebSockets and Server Sent Events directly in HTML, using attributes, so you can build modern user interfaces with the simplicity and power of hypertext</p>
</div>

**What does that mean?**

First, htmx is a single script tag on your app. It allows you to add custom attributes to your HTML elements, that can provide **progressive enhancement** for things like links and forms. <em>Say what?</em> That means an HTML element works as normal, but it can do additional things when hooked up with htmx.

Take a form for example. You have a server-rendered form that makes a POST back to the server, as normal. With htmx, a JS-enabled device can optionally perform that via AJAX and execute some insert or replace behavior on the page.

**What is it NOT?**

Let's compare htmx to other common options, and ponder what is is not. Unlike any JS framework, it does not expect you to rebuild components as a JavaScript template like <a href="https://legacy.reactjs.org/docs/introducing-jsx.html">React's JSX</a>.

It does not mean that, without a JavaScript-enabled device, your application is non-functional. It does not require bundling or compilation. *Not this time Webpack!* And most importantly it does not require you to send JSON objects back and forth to the server.

**Don't send JSON back and forth**

The last point is a really big one for any project with server-generated pages, where templates are written in Buffalo's plush, Go HTML templates, PHP, Ruby, or any other language. Too often it's the case where adding a JS framework means replicating a whole bunch of templates into Javascript. That is a nightmare for consistency and support.

Instead, htmx expects your server to send back fragments of HTML that can be inserted or replaced with no extra logic. So the markup is still done on the server.

If you've used the <a href="https://www.drupal.org/docs/develop/drupal-apis/ajax-api">AJAX API in Drupal</a>, or <a href="https://hotwired.dev/">Hotwired</a>, you will be familiar with some of these concepts. The ideal project for htmx is one with server-generated pages that would benefit from AJAX behaviors.

**So this is modern JQuery?**

No. htmx is not layering JavaScript on top of HTML pages, rather it tries to leverage the original idea of HTML as hypertext to create new interactions. (Their words, not mine.) The API surface area is pretty small compared to jQuery, limiting itself mainly to insert and replace behaviors.

## Project Example - Event Planner App

Let's take a Buffalo project and see what it takes to extend it with htmx. We'll use our Event Planner app from this <a href="/recipe/integrate-vuejs/">recipe on VueJS</a>. Instead of using Vue, we'll modify the server-generated forms to include htmx-ready attributes.

{{< resourceblock >}}
<p>Project code is here: <a href="https://github.com/briwagner/buffalo-vue-integration">Buffalo Vue Integration</a>. Be sure to checkout the <code>ajax-htmx</code> branch.</p>

<p>This was built with Buffalo version 18.14. It was also built with a <a href="https://github.com/briwagner/buffalo-auth">version of the Buffalo Auth plugin</a>.</p>
{{< /resourceblock >}}

### Refactor 1: Apply htmx to a Search Form

**Use-case**: we have a list of events, showing the title and date. This list could be long, so the page also has a text box where users enter search terms to filter the list of events. Buffalo renders the list server-side, like so:

`templates/events/list-remote.plush.html`

{{< codeblock >}}
<div id="eventListRemote" class="mt-3">
  <ul class="list-group">
    <%= for (event) in events { %>
      <li class="list-group-item"><a href="<%= event.ToLink() %>"><%= event.Title %> - <= event.Date.Format("Jan. 02 2006 3:04 PM MST") %></a></li>
    <% } %>
  </ul>
</div>
{{< /codeblock >}}

This will be the default layout when the page loads &mdash; the full list. **Note** we have an ID set on this whole element, which will be used by htmx to perform the replacement. So our steps to refactoring are:

* add an input element (or form) for the text search
* tell htmx how to handle this input
* create a Buffalo route to handle the form-submit
* return a HTML-formatted chunk to replace the original list

So, step one is to add the form. Take note of the `hx-*` attributes, for example the ID from above is added as the `hx-target`.

`templates/events/list-remote.plush.html`

{{< codeblock>}}
<input class="form-control" type="search"
       name="search"
       placeholder="Begin typing to search by event title"
       hx-post="/events/search"
       hx-trigger="input changed delay:500ms, search"
       hx-target="#eventListRemote"
       hx-vals='{"authenticity_token": "<%= authenticity_token %>"}'>
{{< /codeblock >}}

One trick here is to include some extra data in our request, in the form of `hx-vals`. If you're using the default Buffalo setup, then CSRF-protection is in effect, so any POST request without the authenticity-token will fail. In our case, this page is rendered on the server, so we can write that value to the page using Plush syntax.

The "trigger" to htmx is change-detection on the input, with a reasonable delay to allow a user to finish typing something.

Next, we have now committed to taking requests at the `/events/search` route, shown in the `hx-post` attribute above. So let's make that now.

`actions/events.go`

{{< codeblock >}}
type EventSearchForm struct {
	Title string `form:"search"`
}

func EventSearchHandler(c buffalo.Context) error {
	search := EventSearchForm{}
	err := c.Bind(&search)
	if err != nil {
		log.Printf("form bind error %s", err)
		return c.Redirect(301, "/")
	}

	tx := c.Value("tx").(*pop.Connection)
	events := &models.Events{}
	err = events.SearchTitle(tx, search.Title)
	if err != nil {
		log.Printf("error in search %s", err)
		return c.Redirect(301, "/")
	}
	return c.Render(http.StatusOK, r.String(events.ToList()))
}
{{< /codeblock >}}

In Buffalo, the easy way to accept a form submission is to bind to a struct. This route is used to find an event by using a partial title search. We could try to reuse the Event model but it has a bunch of fields that won't be filled by the form. And the form tag (`title`) would have to match our search form payload (`search`). In this case, we use "search" for the input for semantic reasons on the user's browser.

The alternative to reusing the Event struct is to create a single-use struct to handle this search request with the exact properties we need: `EventSearchForm`.

Next, the search piece is a simple SQL query:

`models/event.go`

{{< codeblock >}}
func (e *Events) SearchTitle(tx *pop.Connection, s string) error {
	s = s + "%"
	err := tx.Where("title like ?", s).All(e)
	if err != nil {
		return err
	}
	return nil
}
{{< /codeblock >}}

Final thing to note from our controller above: the return is a string, NOT a full HTML page. Remember that htmx is expecting an HTML fragment to replace the original content on the page; it’s not a full page reload. Let's see how we build that piece, and refactor from the original plush template.

`models/event.go`

{{< codeblock >}}
func (e Event) EventDate() string {
	return e.Date.Format("Jan. 02 2006 3:04 PM MST")
}

func (e Event) ToListItem() string {
	return "<li><a href='/events/" + e.ID.String() + "'>" + e.Title + "</a> &mdash; " + e.EventDate() + "</li>"
}

func (e Events) ToList() string {
	var b strings.Builder
	b.WriteString("<ul>")
	for _, ev := range e {
		b.WriteString(ev.ToListItem())
	}
	b.WriteString("</ul>")
	return b.String()
}
{{< /codeblock >}}

We already had `ToLink` to facilitate getting the relative url as a string. Now our model has even more helper functions to output formatted field content. These new methods do the same work that was happening in our html templates. That seems redundant, but we cannot re-use those templates for htmx, since we only want portions of them. Instead we can refactor those templates to use these methods.

`templates/events/list-remote.plush.html`

{{< codeblock >}}
<div id="eventListRemote" class="mt-3">
  <%= raw(events.ToList()) %>
</div>
{{< /codeblock >}}

The template is much simpler. It may feel strange to have a bunch of model methods that return strings, and put markup in these struct methods. But I think it's fair to apply the mindset of component-based design. In that sense, these methods are used to generate building blocks that can be put together in different patterns for the user. This approach is really key to allow us to satisfy the needs of htmx and the server-side templates, and avoid duplicated code.

### Refactor 2: Modify a Form Submit

Let's talk progressive enhancement!

**Use-case**: we have a form on the event detail page. It shows event details, a list of registered guests, and a form to add another guest. This form takes email and a full-name, email, and when the user clicks submit, the POST is sent with a full-page reload.

The current form uses the Plush `form_for` builder:

`templates/events/_add-guest-form.plush.html`

{{< codeblock >}}
<%= form_for(guest, {action: eventAddGuestPath({id: event.ID})}) { %>
  <%= f.InputTag("Email") %>
  <%= f.InputTag("FullName") %>
  <%= f.SubmitTag("Reserve a spot") %>
<% } %>
{{< /codeblock >}}

If you want to build the form yourself, or see the generated form content, it's here:

{{< codeblock >}}
<form action="/events/3dfb3b98-49d6-4861-b7f7-e272bc99dbd4/add-guest/" id="guest-form" method="POST">

  <input name="authenticity_token" type="hidden" value="3DndjXmBr4ibBPw/uXmYXez8NcqonXKNFk2h+2GsK9GdZ7y+JiJaMPLzphyhLhbuEpv2hJoXdWv2RB1cm2MPng==" />

  <div class="form-group">
    <label class="form-label" for="guest-Email">Email</label>
    <input class="form-control" id="guest-Email" name="Email" type="text" value="" />
  </div>

  <div class="form-group">
    <label class="form-label" for="guest-FullName">Full Name</label>
    <input class="form-control" id="guest-FullName" name="FullName" type="text" value="" />
  </div>

  <input type="submit" value="Reserve a spot" />
</form>
{{< /codeblock >}}

What do we see? That authenticity token is there. The action is set to a path our Buffalo handlers will understand. A traditional html form.

How we do make it work with htmx? Only one change is needed here to add the AJAX behavior, using the `hx-post` attribute.

{{< codeblock >}}
<%= form_for(guest, {
    action: eventAddGuestPath({id: event.ID}),
    hx-post: eventAddGuestPath({id: event.ID})
  }) { %>
  <%= f.InputTag("Email") %>
  <%= f.InputTag("FullName") %>
  <%= f.SubmitTag("Reserve a spot") %>
<% } %>
{{< /codeblock >}}

In the rendered form that looks like this:

{{< codeblock >}}
<form action="/events/3dfb3b98-49d6-4861-b7f7-e272bc99dbd4/add-guest/"
      hx-post="/events/3dfb3b98-49d6-4861-b7f7-e272bc99dbd4/add-guest/"
      id="guest-form"
      method="POST">
...
{{< /codeblock >}}

The `form_for()` helper allows us to pass other attributes to the form, e.g. data-prop-one, hx-post, etc. That `hx-post` attribute takes the same path we use for the normal form submission. (Or add a different one for some custom AJAX behavior.) The default "swap" behavior for htmx is to replace the form contents with the returned HTML fragment. So the form fields are removed and we can replace it with a status message. Let's do that in the controller:

`actions/events.go`

{{< codeblock >}}
// EventAddGuestHandler responds to POST to create add-guest.
func EventAddGuestHandler(c buffalo.Context) error {
	tx := c.Value("tx").(*pop.Connection)
	event := &models.Event{}
	eventID := c.Param("id")

	// Find and validate event.
	err := tx.Find(event, eventID)
	if err != nil {
		log.Printf("event error %s", err)
		return c.Redirect(301, "/")
	}

	// Find and validate guest.
	guest := &models.Guest{}
	err = c.Bind(guest)
	if err != nil {
		log.Printf("form error %s", err)
		return c.Redirect(301, "/")
	}

	foundGuest := &models.Guest{}
	err = tx.Where("email = ?", guest.Email).First(foundGuest)
	if err != nil && !errors.Is(err, sql.ErrNoRows) {
		log.Printf("guest lookup error %s", err)
		return c.Redirect(301, "/")
	}

	if foundGuest.ID.IsNil() {
		// Try to create new guest.
		foundGuest.Email = guest.Email
		foundGuest.FullName = guest.FullName

		err = tx.Create(foundGuest)
		if err != nil {
			log.Printf("error creating guest %s", err)
			return c.Redirect(301, "/")
		}
	}

	res := &models.EventAttendee{}
	res.GuestID = foundGuest.ID
	res.EventID = event.ID

	err = tx.Create(res)
	if err != nil {
		if strings.Contains(err.Error(), "Duplicate entry") {
			c.Flash().Add("warning", "A reservation already exists for that person.")
			return c.Redirect(301, "/events/"+event.ID.String())
		}
		log.Printf("error making reservation %s", err)
		return c.Redirect(301, "/")
	}

	// For HTMX, we return a simple string to render in-place where the form was.
	return c.Render(http.StatusOK, r.String("<div class='alert alert-info'>Reservation complete for "+foundGuest.Email+"</div>"))
}
{{< /codeblock >}}

The only change is at the end, where we return a string, not an html template. This change means our htmx form submission works via AJAX, but the normal form submission won't work anymore. We need to return an html template if the form is returned without AJAX. That's the point of progressive enhancement.

To accomplish that, we need to make the controller detect a normal form submission from one sent by htmx. We have a couple options. Often with Buffalo, we can check the request for `Content-Type: application/json` which indicates that JavaScript is involved. The htmx api allows us to set custom headers with the `hx-headers` attribute. That would work, but the Plush `form_for()` helper doesn't like interprering that JSON string, as it corrupts some of the encoding.

No worry, we can reuse that `hx-vals` atrribute we saw earlier. Let's just add `"htmx" = true`.

`templates/events/_add-guest-form.plush.html`

{{< codeblock >}}
<%= form_for(guest, {
    action: eventAddGuestPath({id: event.ID}),
    hx-post: eventAddGuestPath({id: event.ID}),
    hx-vals: `{"htmx": true}`
  }) { %>
  <%= f.InputTag("Email") %>
  <%= f.InputTag("FullName") %>
  <%= f.SubmitTag("Reserve a spot") %>
<% } %>
{{< /codeblock >}}

Now update the controller to sniff for that value, which only comes through when htmx makes the request.

`actions/events.go`

{{< codeblock >}}
// EventAddGuestHandler responds to POST to create add-guest.
func EventAddGuestHandler(c buffalo.Context) error {
  ...

	err = tx.Create(res)
	if err != nil {
		if strings.Contains(err.Error(), "Duplicate entry") {
			c.Flash().Add("warning", "A reservation already exists for that person.")
			return c.Redirect(301, "/events/"+event.ID.String())
		}
		log.Printf("error making reservation %s", err)
		return c.Redirect(301, "/")
	}

  // Changes here:

	htmx := false
	c.Request().ParseForm()
	for k, _ := range c.Request().Form {
		if k == "htmx" {
			htmx = true
      break
		}
	}
	if htmx {
		// For HTMX, we return a simple string to render in-place where the form was.
		return c.Render(http.StatusOK, r.String("<div class='alert alert-info'>Reservation complete for "+foundGuest.Email+"</div>"))
	}

	c.Flash().Add("info", "Reservation complete for "+foundGuest.Email)
	return c.Redirect(301, "/events/"+event.ID.String())
}
{{< /codeblock >}}

We do a little extra work to parse the form and look for that "htmx" value. Another option is to make a custom struct to handle this request with all the values, Event, Guest and "htmx". In both cases, we know this came from htmx and return a string.

Remember we need to check for htmx-mode early in the flow, so that errors can return either a string, or an HTML page. Sometimes, you see APIs that return the full error page &mdash; not a string &mdash; on an AJAX request. That's not a great experience.

The project code here ignores some of the error cases, so don't make the same mistake!

## Conclusion: Lessons to Using htmx

There are lots of ways to use JavaScript! In my mind, htmx is a new one that uses some old, proven strategies.

1. use the server to render as much as possible of your web pages
1. you can accomplish a lot on a page with only remove and replace commands in the browser
1. serve fragments of HTML &mdash; not JSON &mdash; in your AJAX requests to avoid duplicating code
1. identify the building blocks of your content to understand what the templates should contain

Many web platforms have flexible templating systems that allow you to create "design components" that be used and re-used in various places. With Buffalo, that's partial templates, struct methods that return strings, or <a href="https://gobuffalo.io/documentation/frontend-layer/custom-helpers/">template helpers</a>. Give htmx a try and see what you can build!

