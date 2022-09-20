# Going Live

Imagine this: You're navigating a static website in the browser. You're not scanning around with your eyeballs, though; you're navigating the elements of the document using an input device (say, a keyboard), to tell a screen reader where it should start reading next, and you receive the content as the screen reader speaks it. Good so far? 

Let's complicate the story a little. The website is now a web *app*. It has buttons or controls that may send you to another URL, or they may instigate a change to just part of the DOM. Hopefully, it's laid out so you can still find everything easily, everything behaves like what it claims to be, and nothing is lying about what it is.

That's what we were aiming for when we [cleared the fog and picked low-hanging fruit](https://fly.io/blog/accessibility-clearing-the-fog/) in the LiveBeats UI. In the spirit of our random combo metaphor: We can see the landscape before us clearly. LiveBeats is made up of logical, well labeled parts, with well defined roles for sensible navigation and some semblance of interactivity. Now we're going to stretch a little higher up into the tree and get its real-time features working nicely with my screen reader.

## What lies ahead?

With a clearer map, we're ready to embark on our new, accessible LiveBeats adventure. Not so fast, though. The web was never designed as an application platform. Sure, we've hacked around its many limitations and have *something* resembling interactivity. But those interactions aren't rendered by my screen reader in the same ways that a native app would be. Here are a few ways in which LiveBeats, or almost any other real-time web app, usually doesn't measure up when tested for accessibility.

### Routing doesn't work

In the web's traditional document-centric mode, clicking a link is like pulling a book off the shelf. There's a standard, boring HTTP request that retrieves an HTML payload and serves it up to the browser. It's like exchanging greetings. You almost certainly do it without thinking. Because it's such a straight-forward process, it's easy to have screen readers read out a document title and its content as soon as it loads.

But there *is* no standard for live route updates. Maybe they arrive via `fetch`, or over some more exotic transport. Maybe they replace the entire page or swap out a single element. As you might imagine, throwing standards out the window tends to take accessibility along with them.

### State isn't announced

Route transitions aren't your only issue. The very nature of real-time apps means you've got status updates, chat messages, and other state changes popping in live over the wire. Automatically presenting these changes accessibly is impossible, so you'll almost certainly need to put in extra effort.

### And then the rug gets yanked out from under you

You've got your routes announcing when they've changed. You even have a bunch of fancy alerts speaking as soon as they appear. But changing the DOM is a bit like that fancy party trick where you yank off the tablecloth without taking everything on the table with it.

As you change page content by adding announcements or alerts, be mindful of the focus. If you remove the page element your user is interacting with for instance, the document may no longer be interactive without manually moving focus somewhere. I can't count how many times I navigate to the top of a document to work around having the text or button I was interacting with simply vanish without a trace. We'll get into the complexity of modals next time, but those aren't even easily interactive unless focus is intentionally moved into them.

## Where to?

We'll start with making LiveBeats' routing more accessible. There are a couple ways you might go about doing this.

The easiest way to make routing screen reader accessible is to simply announce the page title or equivalent on transition. This gets the job done but isn't ideal. Can you guess what its biggest flaw is? Hint: we *just* alluded to it.

If you've transitioned to a new page, you've almost certainly swapped out a huge piece of content, probably taking the user's current focus along with it. You might pull a few clever focus tricks as hinted at above, but then you'll likely announce both the page title *and* the content of whatever element you chose to focus on, thus making route transitions unnecessarily chatty. And even so, you've only solved part of the problem.

Another common approach kills two birds with one stone. Instead of presenting the page title, move the user's focus to a standard location with a page title equivalent whenever the route changes. Not only does this announce the transition nicely, but if done well, it puts the user's focus at an ideal place to begin interacting with whatever content just loaded. We'll use this approach for LiveBeats.

But there's no one correct solution for accessible routing. Someone with a cognitive disability might find sudden focus jumps hard to follow. This is why accessibility testing is crucial. Please don't assume that my advice is the final authority on the matter. I'm giving you a solid foundation, not a final destination.

### Decide on a standard

Replacing one standard is usually best done with another. If you've been keeping up, you've got all you need to do this. Remember our `<main/>` element from [last time](https://fly.io/blog/accessibility-clearing-the-fog/)? LiveBeats adopts the convention that the first `<h1/>` child of the page's `<main/>` element is both a suitable route title and focus target. In cases where no `<h1/>` is present, focus directly on `<main/>`. Ideally this never happens, but as developers, we're all too aware how often things that should never happen actually *do*.

### Get focused

Focusing an element on route transition *seems* simple enough. There *is* a `focus()` function, after all. Not so fast, though. As with many of the web's dusty corners, there are a few rough edges you need to be aware of. Let's check out our code for focusing our chosen element, then go through it bit by bit.

```
// Accessible focus handling
let Focus = {
  focusMain(){
    let target = document.querySelector("main h1") || document.querySelector("main")
    if(target){
      let origTabIndex = target.tabIndex
      target.tabIndex = -1
      target.focus()
      target.tabIndex = origTabIndex
    }
  },
...
```

We start by identifying our target element, then ensure that it is non-null. Now things get odd:

```
      let origTabIndex = target.tabIndex
      target.tabIndex = -1
      target.focus()
      target.tabIndex = origTabIndex
```

`focus()` doesn't let you move the caret anywhere you choose. The element has to be capable of accepting focus, meaning a
[tabindex](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/tabindex) attribute must be present. First, we retrieve the original value which, given `<h1/>` isn't typically focusable, is usually `undefined`. Then we set it to -1, implying that we can focus the element programatically but not via tab or shift tab. Finally, we execute the focus, then reset `tabindex` to whatever value it held previously.

We now have a helper function that sets focus where we want. Now we hook it into LiveView so it's called on page transition.

```
let routeUpdated = () => {
  Focus.focusMain()
}

// Accessible routing
window.addEventListener("phx:page-loading-stop", routeUpdated)
```

Notice we hook into the `phx:page-loading-stop` event. This is because we're trying to access the DOM directly but have 2 potential sources of latency:

* Most obviously, the network adds latency by virtue of actually having to receive the data for the new route.
* Further, any page transition animations or effects might result in DOM
  elements not being available when you attempt to focus them.

Fortunately, `phx:page-loading-stop` seems to run our code at the correct time. If it doesn't, though, you may need `setTimeout` or `requestAnimationFrame` shenanigans to introduce artificial delays for transitions or effects.

With this change, live route refreshes automatically move my screen reader to a good page title equivalent. Excellent!

## What's up?

Routing is only part of LiveBeats' story. Real-time apps often have state which changes in response to a variety of events. The most powerful tool for making these state changes accessible is the [ARIA live region](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/ARIA_Live_Regions).

Connectivity is an important aspect of any real-time web app. A boring HTML page will generally continue working whether I'm on 5G or deep underground. By contrast, losing connectivity with LiveBeats means that music stops playing, buttons and links don't work and, if you haven't made your connectivity indicator accessible, screen reader users get very confused.

We'll make LiveBeats' connection status accessible using an ARIA live region. Live regions give you bunches of knobs to control how their content is read,
what types of changes cause the region to be read, whether changes read the entire region or just the difference, etc. For the sake of this example, we'll keep it simple:

```
  def connection_status(assigns) do
    ~H"""
    <div
      id="connection-status"
      class="hidden rounded-md bg-red-50 p-4 fixed top-1 right-1 w-96 fade-in-scale z-50"
      js-show={show("#connection-status")}
      js-hide={hide("#connection-status")}
    >
      <div class="flex">
        <div class="flex-shrink-0">
          <svg class="animate-spin -ml-1 mr-3 h-5 w-5 text-red-800" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" aria-hidden="true">
            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
          </svg>
        </div>
        <div class="ml-3">
          <p class="text-sm font-medium text-red-800" role="alert">
            <%= render_slot(@inner_block) %>
          </p>
        </div>
      </div>
    </div>
    """
  end
```

That's a *lot* of code. Surprisingly, the only bit relevant to accessibility is:

```
          <p class="text-sm font-medium text-red-800" role="alert">
            <%= render_slot(@inner_block) %>
          </p>
```

Remember roles? The [`alert` role](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles/Alert_Role) marks an area of the page as being for important, usually time-sensitive, information. Any content rendered to this element will be read out immediately, cutting off any existing screen reader speech if necessary.

Let me reiterate. "Cutting off any existing screen reader speech if necessary" means exactly that. With great power comes great responsibility. I see web
developers use live regions for *any* section of the page that changes. Carousels and slideshows are the most common abuse. Please only use live regions for changes that benefit me if reported immediately. Incoming alert? Great. Ad banners? Not so much.

Live regions work well for the obvious case of reading content that changes. Buthow do you present updates and alerts that *aren't* associated with an area onthe page? Say someone's post is liked or favorited in your new fancy social networking app. You may never display the text "Alice likes your post," but you might still want to make that accessible. Or maybe you convey something visually, like a greyed-out icon for a dropped connection and want to speak an alert whenever the icon's color changes.

The solution here is an [off-screen live region specifically for announcements](https://stackoverflow.com/questions/50747587/announce-aria-live-text-change-despite-div-being-hidden). One step developers often miss when taking this approach is that of clearing the announcement live region after a reasonable timeout. Without this, the text of. the most recent announcement hangs around in the DOM, usually at the end, and can confuse screen reader users who navigate to the bottom of the page only to find one or more earlier announcements in the page text. Slap a long `setTimeout` to give screen readers enough time to read the text--15 seconds should be sufficient.

## Conclusion

We've done a *lot* to make LiveBeats more screen reader accessible, but there's one piece missing. Uploading songs opens a modal dialog, and these are unfortunately not very accessible without quite a bit of effort. Making these accessible draws on just about every technique we've covered so far, so stick around! And, as always, thanks for taking the time to learn more about
accessibility.
