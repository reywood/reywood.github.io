---
layout: post
title: Publishing Reactive Joins in Meteor
published: True
categories:
    - meteor
tags:
    - javascript
    - meteor
---
Publishing many related documents from different MongoDB collections in your Meteor app can be a hairy problem. You might find yourself calling `Meteor.publish` several times to get all the documents you need. [`Meteor.publishComposite`][publish-composite] was created to solve this problem in a flexible manner.

Let's say you run a site that allows users to post articles and also comment on those articles. Let's also say that, for whatever reason, you don't want to normalize this data so that articles and comments are in separate collections. You'd have a couple of collections defined in your Meteor code that look like this.

{% highlight javascript %}
Articles = new Meteor.Collection("articles");
Comments = new Meteor.Collection("comments");
{% endhighlight %}

An article document might look like:

{% highlight json %}
{
    "_id": "article1",
    "createdAt": ISODate("2014-09-10 01:22:11.405Z"),
    "authorId": "user1",
    "title": "Teen petitions to get cat, lasers in yearbook photo",
    "body": "A student is petitioning for the right ..."
}
{% endhighlight %}

And a comment document might look like:

{% highlight json %}
{
    "_id": "comment1",
    "createdAt": ISODate("2014-09-10 01:24:31.935Z"),
    "authorId": "user2",
    "articleId": "article1",
    "comment": "Ugh, these freakin' teens with their cats and their lasers.",
    "score": 9000
}
{% endhighlight %}

Let's also say that the `authorId` fields in each collection point to a user in `Meteor.users`, and you've chosen not to denormalize the names of the authors into the `articles` and `comments` documents because ... well, who knows. We're talking in hypotheticals here, so save whatever document database best practices rage is building up inside you for another day.

Now, let's say you want to publish a list of the ten newest articles along with the top two comments on each one and the names of the authors of both the articles and the comments. Using `Meteor.publish`, you'd have to create lord only knows how many subscriptions (at least three, I guess). There must be a better way, right? Right?! Yes. Yes, there is a better way, and it's called `Meteor.publishComposite`.

With `Meteor.publishComposite` you can publish this whole hierarchy of documents in a flexible manner. Here's how you would accomplish the convoluted task I've set before you.

{% highlight javascript %}
Meteor.publishComposite("tenMostRecentArticles", {
    find: function() {
        // Let's go ahead and find those top ten articles
        return Posts.find(
            {},
            { sort: { createdAt: -1 }, limit: 10 })
    },
    children: [
        {
            find: function(article) {
                // And let's find each article's author. Now, you might
                // think you could use findOne instead of find here, but
                // you'd be wrong. More on this in a bit.
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
                    find: function(comment) {
                        // And those comments' authors
                        return Meteor.users.find({ _id: comment.authorId });
                    }
                }
            ]
        }
    ]
});

{% endhighlight %}

Looking at this, we can see that the second argument to `Meteor.publishComposite` is an object literal with two properties, `find` and `children`. The `find` property is a function which returns a cursor. This is very important. The `find` function must return a cursor (or `null` if you want to abandon the subscription for some reason). Returning a cursor is what makes this reactive. The `children` property is an array containing yet more object literals with `find` and `children` properties.





[publish-composite]: https://atmospherejs.com/reywood/publish-composite
