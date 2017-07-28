# Conditional Actions

Ok, new challenge! I *only* want this edit button to be visible and accessible if
the user has `ROLE_SUPERADMIN`. This turns out to be a bit complicated... in part
because there are two sides to it.

First, we need truly block *access* to that action... so that a clever user can't
just hack the URL and start editing! And second, we need to actually hide the link...
so that our less-than-super-admin users don't get confused.

## Preventing Action Access by Role

First, let's lock down the actual controller action. How? Now we know two ways:
by overriding the `editAction()` in `UserController` and adding a security check
*or* by adding a `PRE_EDIT` event listener. Let's use events!

Subscribe to a *second* event: `EasyAdminEvents::PRE_EDIT` set to `onPreEdit`:

[[[ code('e070b03d91') ]]]

Once again, I'll hit `Alt`+`Enter` as a shortcut to create that method for me:

[[[ code('1d2e9abb22') ]]]

And just like before... we don't really know what the `$event` looks like. So,
dump it!

Now, as *soon* as I hit edit... we see the dump! Check this out: this time, the
`subject` property is actually an *array*. But, it has a `class` key set to the
`User` class. We can use *that* to make sure we only run our code when we're editing
a user.

In other words, `$config = $event->getSubject()` and `if $config['class']` is equal
to our `User` class, then we want to check security:

[[[ code('95eb5af172') ]]]

Let's call a new method... that we'll create in a moment: `$this->denyAccessUnlessSuperAdmin()`:

[[[ code('da2a9cdac3') ]]]

At the bottom, add that: `private function denyAccessUnlessSuperAdmin()`:

[[[ code('396c5996fc') ]]]

Now... we just need to check to see if the current user has `ROLE_SUPERADMIN`.
How? Via the "authorization checker" service. To get it, type-hint a new argument
with `AuthorizationCheckerInterface`. Hit `Alt`+`Enter` to create and set that property:

[[[ code('62c9316374') ]]]

Then, back down below, `if (!$this->authorizationChecker->isGranted('ROLE_SUPERADMIN')`,
then throw a new `AccessDeniedException()`:

[[[ code('ffb298dfc5') ]]]

Make sure you use the class from the Security component:

[[[ code('f34e0823b5') ]]]

Oh, and don't forget the `new`!

[[[ code('31c22bffa3') ]]]

See, normally, in a controller, we call `$this->denyAccessUnlessGranted()`. When we
do that, *this* is actually the exception that is thrown behind the scenes. In other
words, we're *really* doing the *same* thing that we normally do in a controller.

And... we're done! The service is set to be autowired, so Symfony will know to pass
us the authorization checker automatically. Refresh!

Great news! Access denied! Woohoo! I've never been so happy to get kicked out of
something. Our user does *not* have `ROLE_SUPERADMIN` - just `ROLE_ADMIN` and `ROLE_USER`.
To double-check our logic, open `app/config/security.yml`, and, temporarily, for
anyone who has `ROLE_ADMIN`, also give them `ROLE_SUPERADMIN`:

```yaml
security:
    role_hierarchy:
        ROLE_ADMIN: [ROLE_MANAGE_GENUS, ROLE_ALLOWED_TO_SWITCH, ROLE_SUPERADMIN]
```

Now we should have access. Try it again!

Access granted! Comment-out that `ROLE_SUPERADMIN`.

## Hiding the Edit Button

Time for step 2! On the list page, we need to hide the edit link, unless I have
the role. This is trickier: there's no official hook inside of EasyAdminBundle to
conditionally hide or show actions. But don't worry! Earlier, we overrode the list
template so that we could control *exactly* what actions are displayed. Our new
`filter_admin_actions()` filter lives in `EasyAdminExtension`:

[[[ code('d2ffe9e885') ]]]

And we added logic there to hide the delete action for any published genuses:

[[[ code('72512c9cf7') ]]]

In other words, we added our *own* hook to control which actions are displayed.
We rock!

To hide the edit action, we'll need the authorization checker again. No problem!
Add `public function __construct()` with one argument: `AuthorizationCheckerInterface`.
Set that on a new property:

[[[ code('136d215f44') ]]]

Then, down below, we'll add some familiar code: if `$item instanceof User` and
`!$this->authorizationChecker->isGranted('ROLE_SUPERADMIN')`, then unset the
`edit` action:

[[[ code('1b64373aae') ]]]

Phew! It's not the easiest thing ever... EasyAdminBundle... but it *does* get the
job done!

Except for one... *minor* problem... there is *also* an edit button on the show
page. Oh no! It looks like we need to repeat all of this for the *show* template!

## Controlling the Actions in the show Template

But don't worry! With all our knowledge, this should be quick and painless.

Inside of the bundle, find the show template. And inside it, search for "actions".
Here we go: block `item_actions`. To control the actions, we can do a very similar
thing as the list template. In fact, copy the list template, and paste it as
`show.html.twig`. Because it's in the right location, it should automatically
override the one from the bundle.

Extend that base `show.html.twig` template:

[[[ code('3a64ab5933') ]]]

Before, we overrode the `_list_item_actions` variable and then called the `parent()`
function to render the parent block.

But... that actually won't work here! Bananas! Why not? In this case, the variable
we need to override is called `_show_actions`. And... well... it's set right inside
the block. That's different from `list.html.twig`, where the variable was set *above*
the block. This means that if we override `_show_actions` and then call the parent
block, the parent block will re-override our value! Lame!!!

No worries, it just means that we need to override the *entire* block, and avoid
calling parent. Copy the block and, in `show.html.twig`, paste:

[[[ code('60e4bdd25c') ]]]

Next, add our filter: `set _show_actions = _show_actions|filter_admin_actions`:

[[[ code('4564791f9c') ]]]

Remember, we need to pass the entity object as an argument to `filter_admin_actions`...
and that's another difference between show and list. Since this template is for a
page that represents one entity, the variable is not called `item`, it's called
`entity`.

As crazy as that looks, it should do it! Hold you breath, do a dance, and refresh!

Hey! No edit button! Go back to `security.yml` and re-add `ROLE_SUPERADMIN`:

```yaml
security:
    role_hierarchy:
        ROLE_ADMIN: [ROLE_MANAGE_GENUS, ROLE_ALLOWED_TO_SWITCH, ROLE_SUPERADMIN]
```

Refresh now. Edit button is back. And we can even use it. One of the least-easy
things in EasyAdminBundle is now done!
