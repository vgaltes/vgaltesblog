---
title: 'Beyond Event Storming'
date: '2016-12-11'
categories:
- Event Storming
tags:
- Business analisys
- DDD
---
On 1st and 2nd of December, the last edition of [Conferencia Agile Spain](http://cas2016.agile-spain.org/) was held in Vitoria. I was part of the oragnisation and both as a organiser and as attendee I think it was *awesome* (post with some of the internals will come soon). I held a workshop about [Event Storming](http://ziobrando.blogspot.co.uk/2013/11/introducing-event-storming.html) there, which went great. [Chris Matts](https://twitter.com/papachrismatts) did the opening keynote and he attended my workshop too. In the middle of the workshop he approached to me and told me: "If you want, after the workshop I can explain you how can you join Event Storming and [Real Options](https://www.infoq.com/articles/real-options-enhance-agility)." Of course! I said.

So, along with [Silvia Calvet](https://twitter.com/silviacalvet) and another guy who I can't remember his name (sorry about that mate!), Chris did a one hour master class about Risk Management and Business Analysis. For me it was the best session of the conference and it wasn't in the agenda :-)

The summary of the master class could be: "[Event Storming](http://eventstorming.com/) is a great tool when you know what do you need to do, but there's a bunch of very important activities that you have to make before getting to this point." And after the explanation (and with my limited experience in this part of the development of a product) I agree. [Event Storming](https://leanpub.com/introducing_eventstorming) a great tool even for discovering hidden points in your domain and your solution, but its position is further down in the list of activities to do when releasing a product. It would be great to know [Alberto Brandolini](https://twitter.com/ziobrando)'s opinion about this point.

So, what [Chris Matt](https://theitriskmanager.wordpress.com/)'s think the process should be? Let me explain you with the example he used. Bear in mind that he did a basic summary of all of his experience, I'm sure there's a lot more. 

Imagine that you want to produce something that helps you to make tea (a teabag, basically). First of all you need to study the needs of your users? Why they need a teabag? To make and drink some tea? Or maybe to change the colour of their curry?

![teabag](/images/teabag.png)

Let's say we want to drink some tea. What are the needs we want to fulfil? Could be a bunch of them like:

 - because the user wants to be warm
 - because the user is thirsty
 - because it's a social activity
 - because it's a habit

Those needs can be different for different users. We need to study which segments of users we have because maybe we want to focus in one of them:

 - commuters
 - friends
 - old ladies

And finally we have to study which business value we want to get from our product:

 - money 
 - engagement

![Needs, business value and segments](/images/needs.png)

As you can see we're not inventing any data here. We're just collecting what our users want. We're not relying on someone having a magic idea.

With this data we can construct a matrix. In one axis we can put the different segments we have, in another one we can put the different business value and in the different cells we can put how many insights we have for that pair. With that information we can see where we need more data.

So we now have lots of data and we want to develop the next killing feature of our app. How can we know which one we have to choose?

For each insight we have we need to specify which Business Value (PO), Need (UX researcher) and Segment (UX researcher and data analyst) that insight is for. With that, we can start doing a design brief. That means that we're going to explore which options do we have, how the application will be and with the help of different types of testing decide if the epic worths the effor. This is obviously an iterative process. If the epic worths the effor when now start to explore it in more details. And here is where EventStorming comes to action. It can be a great tool to transition from this epic we want to implement, to the different stories we need to implement.

![Design](/images/design.png)

Chris explained another way to go from the epic to the stories. For him, the inventor along with [Dan North](https://www.twitter.com/tastapod) of the Given-When-Then, it's quite natural to specify the stories in this format and he developed a technique that I really liked a lot. 

Imagine that your developing an [Imdb](http://www.imdb.com) for music and you want to provide a feature for the segment fans where you can see the places that are important for a music star. Let's then start with the most important outcome for you

    Then displays the map
    and my location
    and the music star sites

What have the user to do to achieve this outcomes?

    When I'm near a music star site

Which are the context of this action?

    Given I'm a fan of a music star
    and I'm GPS enabled
    and there are sites for that music star

And there you are! You have your Given-When-Then backward developed. Two important addendums to this technique:
 - The context in a Given-When-Then can be mocked. That will be important when you want to develop a quick prototype of the feature.
 - The Given of your first Given-When-Then can be the Then of your previous feature. In this example, we can end up writing something like

    Given blah blah blah
    When blah blah blah
    *Then I'm a fan of a music star*

And that is super cool and I think in some way links with the EventStorming idea, when that can be a domain event and we can think from that which are the options we have to produce that event.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">&quot;why modeling with events?&quot; &quot;because events are observable evidence, testable &amp; create options for activities that lead to them&quot; <a href="https://twitter.com/tastapod">@tastapod</a></p>&mdash; Bouillier Clément (@clem_bouillier) <a href="https://twitter.com/clem_bouillier/status/798586783957139456">November 15, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Maybe for you this is something quite trivial and known, but for me was something exciting to discover. There is a well known process that helps us to discover what is the most valuable feature to deliver. I think that's quite related to the work that Fatma and Nazli explained ThoughtWorks is doing in some projects. I will put the link tho their talk here.

This conversation with Chris was eye-opening for me. It opened a door into a world I didn't know about it and that I find fascinating. To get deeper in this idea, I attended the workshop about [Design Sprints](http://www.gv.com/sprint/) from Silvia Calvet and [Gastón Valle](https://gastonvalle.com/2016/12/06/mi-primera-cas-como-ponente/) and I found it amazing. I really want to do this kind of things as soon as I can and study more about this.