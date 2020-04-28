---
layout: layouts/post.njk
title: Handling Slow Cascading Deletes In Rails
date: 2020-02-27T10:29:57.677Z
tags:
  - Engineering
---
> Daddy Warbucks : You lock the orphans in the closet.
>
> Miss Hannigan : They love it!

---------

### TL;DR 

If you're using a development framework to interact with data (Rails, Django, Express, whatever), you've most likely had to make smart choices on how cascading deletes work in your system. And often in large systems, you're forced to make a compromise...

You could have your app and framework manage all those deletes keeping close control over validation and object lifecycles, which is ideal but often poor in performance for large records sets. Or, you could have your database handle the cascading deletes, but then you lose things like your lifecycle hooks.

So today, for Rails, we're releasing a small but useful utility that provides an alternative (and in some cases, better) way to do cascading deletes/destroys. It's called [**Miss Hannigan and you can find it here.**](https://github.com/sutrolabs/miss_hannigan)

-------------

To quickly catch beginners up, Rails has some great tooling to deal with parent-child relationships using `has_many`:

```ruby
  class Parent < ApplicationRecord
      has_many :children
  end
```

By default, what happens to `children` when you delete an instance of Parent? Nothing. Children just sit tight or in our more typical vernacular, they're orphaned.

To cleanup those orphaned records, you consider two Rails options we'll explain below: destroy the children, or delete the children. Both options present some problems.

### dependent: :destroy

Destroying the children is ideal. You do that by setting `dependent: :destroy` on the has_many relationship. Like so:

```ruby
class Parent ApplicationRecord
    has_many :children, dependent: :destroy
end
```

Rails, when attempting to destroy an instance of the Parent, will also iteratively go through each child of the parent calling destroy on the child. The benefit of this is that any callbacks and validation on those children are given their day in the sun. If you're using a foreign_key constraint between Parent -> Child, this path will also keep your DB happy. (The children are deleted first, then the parent, avoiding the DB complaining about a foreign key being invalid.)

But the main drawback is that destroying a ton of children can be time consuming, especially if those children have their own children (and those have more children, etc.).
If you're on a hosting platform like Heroku, which caps your max request length at 30s, your deletes won't even work as they'll be rolled back following a timeout error.

So, many of us reach for the much faster option of using a `:delete_all` dependency.

### dependent: :delete_all

Going this route, Rails will delete all children of a parent in a single SQL call without going through the Rails instantiations and callbacks.

```ruby
class Parent ApplicationRecord
    has_many :children, dependent: :delete_all
end
```

However, `:delete_all` has plenty of problems because it doesn't go through the typical Rails destroy.

For example, you can't automatically do any post-destroy cleanup (e.g. 3rd party API calls) when those children are destroyed.

And you can't use this approach if you are using foreign key constraints in your DB:

![](https://github.com/sutrolabs/miss_hannigan/blob/master/foreign_key_error_example.png?raw=true)

Another catch is that if you have a Parent -> Child -> Grandchild relationship, and it uses `dependent: :delete_all` down the tree, destroying a Parent, will stop with deleting the Children. Grandchildren won't even get deleted/destroyed.

------------

Here at [Census](http://getcensus.com) this became a problem. We have quite a lot of children of parent objects. And children have children have children... We had users experiencing timeouts during deletions.

Well, we can't reach for `dependent: :delete_all` since we have a multiple layered hierarchy of objects that all need destroying. We also have foreign_key constraints we'd like to keep using.

So what do we do if neither of these approaches work for us?

We use an "orphan then later purge" approach. Which has some of the best of both `:destroy` and `:delete_all` worlds.

dependent has a nifty but less often mentioned option of `:nullify`.

```ruby
class Parent < ApplicationRecord
    has_many :children, dependent: :nullify
end
```

Using `:nullify` will simply issue a single UPDATE statement setting children's parent_id to NULL. Which is super fast.

This sets up a bunch of orphaned children now that can easily be cleaned up in an asynchronous purge.

And now because we're destroying Children here, the normal callbacks are run also allowing Rails to cleanup and destroy GrandChildren.

Fast AND thorough.

So we wrapped that pattern together into a utility we're releasing to everyone today. [**We call it Miss Hannigan.**](https://github.com/sutrolabs/miss_hannigan) (Destroyer of Orphans. Miss Hannigan sounded less Game of Thrones)

It's a small piece of kit, but it gives you a new ability with `has_many` relationships. You can now add a `:nullify_then_purge` dependency. Like so:

```ruby
class Parent < ApplicationRecord
    has_many :children, dependent: :nullify_then_purge
end
```

This will do the nullify and asynchronous purge (using ActiveJob) for you out of the box.

Another nice benefit, is that we use batch destroys of the children lowering the memory size of destroying a lot of children (a feature that's been [dead on the vine for Rails Core](https://github.com/rails/rails/issues/22510) for awhile).

------------------

## Alternatives

It's worth noting there are other strategies to cleanup data when parent objects are deleted. For example, you could allow your DB (if it supports it) to handle its own cascading deletes. With Rails you would do:

```ruby
add_foreign_key "children", "parents", on_delete: :cascade
```

Using that on Postgres would have children automatically delete when a parent is deleted. But that removes itself from Rails-land where we have other cleanup hooks and tooling we'd like to keep running.

Another alternative would be to use a pattern like acts_as_paranoid to "soft delete" a parent record and later destroy it asynchronously.

-----------------------

We hope you find it handy. It's not a big project, so you could add this in yourselves, but this makes it just a tad slicker. Would love to hear if you get a chance to use it or how you'd improve it.

[**You can find Miss Hannigan here.**](https://github.com/sutrolabs/miss_hannigan)