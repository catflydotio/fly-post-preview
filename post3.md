# Going Live

Imagine this: You're navigating a static website in the browser. You're not
scanning around with your eyeballs, though: you're navigating the elements of
the document using say, a keyboard. You receive the content as the screen reader
speaks it. Good so far? 

Let's complicate the story a little. The website is now a web *app*. It probably
has more interactive elements, like buttons and forms, because we're (probably)
doing more than reading content from top to bottom now. Hopefully it's laid out
so you can still find everything easily, everything behaves like what it claims
to be, and nothing is lying about what it is.

That's what we were aiming for when we [cleared the fog and picked low-hanging
fruit](https://fly.io/blog/accessibility-clearing-the-fog/) in the LiveBeats UI.
In the spirit of our random combo metaphor: we can see the landscape before us
clearly. LiveBeats is made up of logical, well-labeled parts, with well-defined
roles for sensible navigation and some semblance of interactivity. 

We're ready to embark on our new, accessible LiveBeats adventure. But we're not
in the clear yet. Now we stretch a little higher up into the tree and get its
real-time features working nicely with screen readers and other assistive
technologies.

## What lies ahead? 

The web was never designed as an application platform. We've hacked around its
limitations to get something resembling interactivity, but we're still working
with something that believes, in its heart of hearts, that it's a document.
Interactions still aren't rendered by my screen reader in the same ways they
would be in a native app.

Here are a few ways in which real-time web apps usually don't measure up when
tested for accessibility.

* Route changes aren't announced.
* State changes aren't spoken by screen readers, and may not be obvious to
  magnifiers and other assistive technologies.
* The element the user's interacting with might vanish when a page transitions.

In the web's traditional document-centric mode, clicking a link has standard
consequences. It's a like pulling a book off the shelf. The browser makes a
standard HTTP request and the server sends back an HTML payload to the browser.
The screen reader then presents the title of the page when it loads. You grab a
book, the screen reader confirms your choice. There's nothing surprising here.

Route changes in web apps aren't constrained to a standard behavior. They're
generally not achieved by an HTML request and response, which means that the URL
displayed in the browser, and all or part of the DOM, can get swapped without a
screen reader noticing. When standards go out the window, accessibility often
goes with them.

Live state changes are another thing that can sneak past a screen reader in a
real-time app. Status updates, chat messages, etc. pop in live over the wire,
and different updates may need to be handled differently. The screen reader has
no context for what these document changes actually mean, so they'll need
individual attention to ensure they're announced in correctly. Also, while it
may be OK for some state changes to assertively announce themselves, others
should be more polite and not interrupt existing speech.

However you're changing the DOM, be mindful of the focus. Live-patching the DOM
is kind of like yanking a tablecloth without taking everything on the table with
it. Nice! You didn't so much as spill a drink! But if a guest was admiring the
fabric or dabbing at a drop of gravy, you've yoinked the thing they were doing
out from under them.

If you remove the page element your user is interacting with, the document may
no longer be interactive without manually moving focus somewhere. I can't count
how many times I've navigated to the top of a document to work around having the
text or button I was interacting with simply vanish. 

Route changes and state announcements can both mess with focus if you don't take
steps to prevent it. Don't even talk to me about modals. (Actually, we'll get
into the complexity of modals next time. Spoiler: those aren't even easily
interactive unless focus is intentionally moved into them.)

## Where to? (Routing)

Let's turn our attention to how this plays out in LiveBeats. LiveBeats is a
Phoenix LiveView app, using WebSockets for client-server communication. Since we
don't have a standard to tell us how to get this to work with a screen reader,
we're free to choose a "standard" pattern that suits us. And we should.

If we've transitioned to a new page, we've almost certainly swapped out a huge
piece of content, probably taking the user's current focus along with it. We
could emulate the HTML standard and have the page title announced on a route
transition. But we'd still need to manage the focus, so we'd likely announce
both the page title and the content of whatever element we chose to focus on,
making route transitions unnecessarily chatty.

There's an approach that kills two birds with one stone: Instead of presenting
the page title, move the user's focus to a standard location with a page title
equivalent whenever the route changes. Not only does this announce the
transition nicely, but if done well, it puts the user's focus at an ideal place
to begin interacting with whatever content just loaded. We'll use this approach
for LiveBeats.

I want to stop here and emphasize that here's no one correct solution for
accessible routing. Someone with a cognitive disability might find sudden focus
jumps hard to follow. This is why accessibility testing is crucial. Please don't
assume that my advice is the final authority on the matter.

In LiveBeats, the content that changes when we visit a new route [is
encapsulated within the `<main/>`
element](https://fly.io/blog/accessibility-clearing-the-fog/). Then LiveBeats
adopts the convention that the first `<h1/>` child of `<main/>` is both a
suitable route title and focus target. As a fallback, if no `<h1/>` is present,
focus goes directly to `<main/>`. Ideally this never happens, but as developers,
we're all too aware how often things that should never happen actually *do*.

Let's check out the LiveBeats code for focusing our chosen element. We'll use
the
[`HTMLElement.focus()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/focus)
method:

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

We start by finding our target element, then ensure that it is non-null. 

Then we have to deal with the fact that `focus()` doesn't let you move the caret
to whatever element you want. The element must be capable of accepting focus,
meaning either the element is explicitly focussable as per the HTML
specification, or a
[tabindex](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/tabindex)
attribute must be present. So we do this little dance:

```
      let origTabIndex = target.tabIndex
      target.tabIndex = -1
      target.focus()
      target.tabIndex = origTabIndex
```

We save the original value of `tabindex` which, given `<h1/>` isn't typically
focusable, is usually `undefined`. Then we set it to `-1`, meaning that we can
focus the element programmatically but not via tab or shift-tab. Finally, we
execute the focus, then reset `tabindex` to whatever value it held previously.

We now have a helper function that sets focus where we want. Now we hook it into
LiveView so it's called on page transition.

```
let routeUpdated = () => {
  Focus.focusMain()
}

// Accessible routing
window.addEventListener("phx:page-loading-stop", routeUpdated)
```

There are two potential sources of delay before our DOM elements are available
to focus:

* Most obviously, the network adds latency by virtue of actually having to
  receive the data for the new route.
* Further, page transition animations or effects might result in DOM elements
  not being available when you try focussing them.

In LiveView, the `phx:page-loading-stop` event fires when the DOM is finished
loading after a live redirect or DOM patch. We hook into this event to run our
code at the correct time. If it doesn't, though, you may need `setTimeout` or
`requestAnimationFrame` shenanigans to introduce artificial delays for
transitions or effects.

With this change, live route refreshes automatically move my screen reader to a
good page title equivalent. Excellent!

## What's up? (State changes)

Real-time apps often have state which changes in response to a variety of
events. Some of these we'll want to know about as they happen. The most powerful
tool for making these state changes accessible is the [ARIA live
region](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/ARIA_Live_Regions). 

Live regions give us knobs to control [how assertively their content is
read](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-live),
[what types of changes cause the region to be
read](https://fly.io/blog/accessibility-clearing-the-fog/#role-with-it-but-not-too-far),
[whether changes read the entire region or just the
difference](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-atomic),
[etc.](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-busy)

Remember
[roles](https://fly.io/blog/accessibility-clearing-the-fog/#role-with-it-but-not-too-far)?
The [`alert`
role](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles/Alert_Role)
is a shorthand for live region settings for important, usually time-sensitive,
information. Changes to such an area will be read out immediately, cutting off
any existing screen reader speech if necessary.

"Cutting off any existing screen reader speech if necessary" means exactly that.
With great power comes great responsibility. I see web developers use live
regions for *any* section of the page that changes. Carousels and slideshows are
the most common abuse. Please only use live regions for changes that absolutely
need to be reported immediately. Incoming alert? Great. Ad banners? Not so much.
And yes, I've encountered ad banners using live regions.

One thing I do want to hear about immediately is when my real-time web app loses
connectivity. A boring HTML page will generally continue working whether I'm on
5G or deep underground. By contrast, losing connectivity with LiveBeats means
that music stops playing, buttons and links don't work and, if you haven't made
your connectivity indicator accessible, screen reader users get very confused.

Let's make LiveBeats' connection status accessible using `role="alert"`. Here's
the only part of the [connection status
indicator](https://github.com/fly-apps/live_beats/blob/6b02cfc614aaf1f7a5ebc595c435bf62a65f5bcb/lib/live_beats_web/live/live_helpers.ex#L27)
relevant to accessibility:

```
<p class="text-sm font-medium text-red-800" role="alert">
  <%%= render_slot(@inner_block) %>
</p>
```

Picking the best live region settings was most of the work! An incoming chat
message could probably afford to [wait for other announcements to
finish](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-live),
and maybe you don't even need to know every time another user visits or leaves
your playlist. In a music-player app, you really don't want too many status
announcements interrupting the tunes.

## Ephemeral alerts

Live regions work well for the obvious case of reading content that changes
somewhere in the DOM. But how do you present updates and alerts that *aren't*
associated with an area on the page? Say someone's post is liked or favorited in
your new fancy social networking app. You may never display the text "Alice
likes your post," but you might still want to make that accessible. Or maybe you
convey something visually with a symbol but no text; say, a greyed-out icon for
a dropped connection; and want to speak an alert whenever the icon's color
changes.

The solution here is an [off-screen live region specifically for
announcements](https://stackoverflow.com/questions/50747587/announce-aria-live-text-change-despite-div-being-hidden).

One step developers often miss with this kind of announcement is to clear the
live region after a reasonable timeout&mdash;it's still hanging around the DOM
after it's spoken, usually at the end, waiting to surprise screen reader users
when they navigate to the bottom of the page. Again, given how often we have to
work around loss of focus issues by moving to the top or bottom of the page,
this is a lot more common than you might expect. And just because a DOM element
is hidden to *you* doesn't mean assistive technologies can't see it.

Slap a long `setTimeout` to give screen readers enough time to read the text; 15
seconds should be sufficient. When the timeout expires, clear the text of the element containing the ephemeral announcement, or remove it entirely.

## Conclusion

We've done a *lot* to make LiveBeats more screen reader accessible, but there's
one piece missing. Uploading songs opens a modal dialog, and these are,
unfortunately, not very accessible without quite a bit of effort. Making these
accessible draws on just about every technique we've covered so far, so stick
around! And, as always, thanks for taking the time to learn more about
accessibility.
