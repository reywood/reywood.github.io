---
layout: post
title: Publishing to an Alternative Client-side Collection in Meteor
disqus_identifier: Hdz5OFsgsTTdzYrwZdBGvYL65d8ofYCxYGIZCbzM0
published: True
categories:
    - meteor
tags:
    - javascript
    - meteor
    - publishComposite
---
When publishing a bunch of documents to the client with `Meteor.publishComposite`, we sometimes end up with a perplexing problem on the client side of things. How do we figure out which published documents were intended for which purpose?

*Note: This post assumes some familiarity with Meteor.publishComposite. If you haven't used Meteor.publishComposite before, I recommend reading my previous post, [Publishing Reactive Joins in Meteor]({% post_url 2014-09-12-publishing-reactive-joins-in-meteor %}), first.*

Maybe this problem can be better illustrated through example. Say we are publishing a list of the top ten articles on our site and want to include some related articles for each one. These related articles will be published via the `children` option of `Meteor.publishComposite`. Normally, all the articles, both parents and children, would end up in the same collection on the client (e.g., Articles) making it difficult to determine which ones are the top ten articles and which are the related articles. What if, instead, we could publish the related articles into a separate client-side collection (e.g., RelatedArticles)? That would be amazingly helpful, wouldn't it? Well, it's a good thing you're reading this blog post, because you can do exactly that with `Meteor.publishComposite` using the `collectionName` option.

The `collectionName` option instructs `Meteor.publishComposite` to tell the client a little lie and say that whatever documents come out of the cursor returned by the `find` option came from a different collection. Here's an example to demonstrate.

{% highlight javascript %}
// The Articles collection is present on both client and server and is
// backed, as usual, by MongoDB.
Articles = new Meteor.Collection("articles");

if (Meteor.isServer) {
    // We create our publication on the server. Note the "collectionName" option.
    Meteor.publishComposite("topTenArticles", {
        find: function() {
            return Articles.find({}, { sort: { score: -1 }, limit: 10 });
        },
        children: [
            {
                // Here we specify the client-side collection name we would
                // like to publish these records to.
                collectionName: "relatedArticles",

                find: function(article) {
                    return Articles.find({ relatedTo: article._id }, { limit: 2 });
                }
            }
        ]
    });
}

if (Meteor.isClient) {
    // The RelatedArticles collection is only present on the client. It's contents
    // are not persisted anywhere, but instead come from the publication.
    RelatedArticles = new Meteor.Collection("relatedArticles");

    // Subscribe to the publication, perhaps in our router's waitOn method
    Meteor.subscribe("topTenArticles");

    // The Articles collection will contain just the top ten articles
    var topTenArticles = Articles.find();

    // The client-side RelatedArticles collection will contain the related articles
    var relatedArticles = RelatedArticles.find();
}
{% endhighlight %}
