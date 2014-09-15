---
layout: post
title: Publishing Reactive Joins in Meteor
disqus_identifier: XLVgg3OFMbxWiumRMTX6x4mAXyC1rMf6Ca32k7J5BI
published: True
categories:
    - meteor
tags:
    - javascript
    - meteor
---
Publishing many related documents from different MongoDB collections in your [Meteor][meteor] app can be a hairy problem. You might find yourself calling `Meteor.publish` several times to get all the documents you need pushed to the client. The [reywood:publish-composite][publish-composite] package was created to solve this problem in a flexible manner. It exposes one new function called `Meteor.publishComposite`.

For the purposes of this post, I'm going to assume that you're already familiar with [Meteor][meteor], and that you're capable of following the simple installation instructions on the [reywood:publish-composite][publish-composite] page.

### The basics

Let's say you run a site that allows users to post articles and also comment on those articles. Let's also say that, for whatever reason, you don't want to normalize this data so that articles and comments are in separate collections. You'd have a couple of collections defined in your Meteor code that look like this.

{% highlight javascript %}
Articles = new Meteor.Collection("articles");
Comments = new Meteor.Collection("comments");
{% endhighlight %}

An article document might look like:

{% highlight json %}
{
    "_id": "article1",
    "createdAt": ISODate("..."),
    "authorId": "user1",
    "title": "Teen petitions to get cat, lasers in yearbook photo",
    "body": "A student is petitioning for the right ..."
}
{% endhighlight %}

And a comment document might look like:

{% highlight json %}
{
    "_id": "comment1",
    "createdAt": ISODate("..."),
    "authorId": "user2",
    "articleId": "article1",
    "comment": "Ugh, these freakin' teens with their cats and their lasers.",
    "score": 9000
}
{% endhighlight %}

Let's also say that the `authorId` field in each document points to a user in `Meteor.users`, and you've chosen not to denormalize the names of the authors into the `articles` and `comments` documents because ... well, who knows. We're talking in hypotheticals here, so save whatever document database best practices rage is bubbling up inside you for the real world.

Now, let's say you want to publish a list of the ten newest articles along with the top two comments on each one and the names of the authors of both the articles and the comments. Using `Meteor.publish`, you'd have to create lord only knows how many subscriptions (at least three, I guess). There must be a better way, right? Right?! Yes, there is a better way, and it's called `Meteor.publishComposite`.

With `Meteor.publishComposite` you can publish this whole hierarchy of documents in a flexible manner with one publication. Here's how you would accomplish the convoluted task I've set before you.

{% highlight javascript %}
Meteor.publishComposite("tenNewestArticles", {
    find: function() {
        // Let's go ahead and find those ten newest articles
        return Articles.find({}, {
            sort: { createdAt: -1 },
            limit: 10
        });
    },
    children: [
        {
            find: function(article) {
                // And let's find each article's author. Now, you might
                // think you could use findOne instead of find here, but
                // you'd be wrong. More on this in a bit. Note we get an
                // article document passed in as an argument.
                return Meteor.users.find({ _id: article.authorId });
            }
        },
        {
            find: function(article) {
                // And the top two comments on each article
                return Comments.find(
                    { articleId: article._id },
                    { sort: { score: -1 }, limit: 2 });
            },
            children: [
                {
                    find: function(comment, article) {
                        // And those comments' authors. Hey! We have two
                        // args this time! We get all the documents going
                        // up the hierarchy passed in. Nearest parent
                        // gets passed in first.
                        return Meteor.users.find({ _id: comment.authorId });
                    }
                }
            ]
        }
    ]
});
{% endhighlight %}

The first argument to `Meteor.publishComposite`, as you guessed, is the name of the publication. Nothing new there. The the second argument is an object literal with two properties, `find` and `children`.

The `find` property is a function which returns a cursor. This is very important. The `find` function **must** return a cursor which is why we use `Meteor.users.find` instead of `Meteor.users.findOne` to find the authors. Returning a cursor is what makes this whole thing reactive. Also worth noting, `find` functions are run in the context of your publication, so you'll have access to properties like `this.userId` as you would in a normal publication.

The `children` property is an array containing yet more object literals with `find` and `children` properties. The `find` function for a child is called for each document returned in the parent `find`'s cursor with that parent document as the argument. So for each article returned in the topmost `find`, the two child `find` functions are called to find the author and the top two comments. The `children` property is optional at every level in the hierarchy.

Remember when I said that `find` **must** return a cursor. Well, I lied ... a little. There will be times when you don't want to publish anything because some check condition failed. Maybe you only want to publish something if the user is logged in. In this case, your `find` function can return a falsy value (`undefined`, `null`, etc) and the publication will short circuit. That might look something like this.

{% highlight javascript %}
Meteor.publishComposite("membersOnly", {
    find: function() {
        if (!this.userId) {
            return;
        }

        return MyPreciousssss.find({ ... });
    },
    children: [
        // ...
    ]
});
{% endhighlight %}


### Reactivity and duplicate data

`Meteor.publishComposite` will publish additions, changes, and removals affecting your documents the same way `Meteor.publish` does. But what would happen if the `authorId` field of an article changed? Drat, now the wrong author document from `Meteor.users` is published to the client. Not to worry, `Meteor.publishComposite` will see that the article has changed and check to see if any of it's child documents need to be added, changed, or removed. The old author will be removed and the new one added.

This leads to another interesting question. We've published a bunch of articles and comments, each with an associated author. What if multiple articles and/or comments were penned by the same author? Are we going to send that author's document to the client multiple times. Luckily for us, `Meteor.publishComposite` is smart enough to know when it has already sent a document to the client and won't repeat itself.


### Passing arguments

Now I know what you're thinking "Wait a minute! I can pass arguments to a normal publication. What gives?" `Meteor.publishComposite` can also accept a function as its second argument instead of an object literal. This function just has to return the object literal that we described above. Maybe you want to publish all the articles by a particular author. Let's go ahead and do that.

{% highlight javascript %}
Meteor.publishComposite("articlesByAuthor", function(authorId) {
    return {
        find: function() {
            return Articles.find({ authorId: authorId });
        },
        children: [
            // ...
        ]
    };
});
{% endhighlight %}

Pretty straighforward, yes?


### A word about flexibility

So far, our examples have generally mapped a primary key in one record to a foreign key in another record, much like you would with a standard ORM. `Meteor.publishComposite` aims to give you more flexibility than that.

Say you want to publish an article for display. After displaying the article, you'd like to provide links to the articles that were created immediately before and after. Not too difficult.

{% highlight javascript %}
Meteor.publishComposite("articleAndFriends", function(articleId) {
    return {
        find: function() {
            return Articles.find({ _id: articleId });
        },
        children: [
            {
                find: function(article) {
                    // Get the previous article
                    return Articles.find(
                        { createdAt: { $lt: article.createdAt } },
                        { sort: { createdAt: -1 }, limit: 1 });
                }
            },
            {
                find: function(article) {
                    // Get the next article
                    return Articles.find(
                        { createdAt: { $gt: article.createdAt } },
                        { sort: { createdAt: 1 }, limit: 1 });
                }
            }
        ]
    };
});
{% endhighlight %}

Get as creative as you like with your relationships.


### Alright already, let's wrap this up

So there you are. You've published a bunch of documents from various collections and in a somewhat sensible manner. Feels good, doesn't it?


[meteor]: https://www.meteor.com/
[publish-composite]: https://atmospherejs.com/reywood/publish-composite
