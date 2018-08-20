---
title: 'Cowboys'
date: '2017-03-14'
categories:
- Teamwork
tags:
- Teamwork
- Antipatterns
---
According to the Wikipedia, a [cowboy](https://en.wikipedia.org/wiki/Cowboy) (and a cowgirl) is an animal herder who tends cattle on ranches in North America, traditionally on horseback, and often performs a multitude of other ranch-related tasks. They were quite famous in the 19th century and we can still find them in ranches.

![original cowboy](/images/cowboy.jpg)

Hollywood adapted the cowboy lifestyle to create some stereotypes, both positive and negative. But, in general, a cowboy was a guy with a gun killing people, either because they were bad guys or because they were defending a good cause.

![Hollywood cowboy](/images/hollywood-cowboy.jpg)

As we've already said, there are still cowboys nowadays, either on ranches or in your development team. What is a cowboy in your development team? They use to share some common characteristics or behaviours:
 - They don't like to pair program with anyone.
 - They have a silo of knowledge that they are not keen to share.
 - They make changes in Production boxes by hand.
 - They only participate in team meetings to patronise the rest of the team.
 - They write long emails (or slack messages). Again, patronising the team.
 - They don't ask the team for consense before making decisions that can affect the team.
 - They are often "good" developers.

And we can continue with that list forever. 

We can differentiate two types of development cowboys:
 - The [Lucky Luke](http://www.lucky-luke.com/en/) type: They are good guys, they don't kill anybody but they can destroy a project. They don't really know they are cowboys and within a good environment, they can change. A good example is [Brent](http://devopsdictionary.com/wiki/Brent) from the book [The Phoenix project](https://www.amazon.co.uk/Phoenix-Project-DevOps-Helping-Business-ebook/dp/B00AZRBLHO/)
 - The [Billy The Kid](http://www.aboutbillythekid.com/) type: They like to be cowboys and they think cowboys are great. They can kill people, teams and projects. They especially like to hurt the project so they can demonstrate they are brilliant and they can "heal" it. They don't like teamwork and they will show it as often as they can. A good example can be the guy who is sitting next to you.

There a couple of characteristics that make cowboys appealing to some kind of managers (those managers that you don't want to work with): they are often "good" developers and they make changes directly in the production boxes. Some managers like this kind of hacks to overcome some problem. Even though there is no rush, they want to get rid of that problem quickly, so there's nothing better than someone that hacks something in the production box, or make a quick change in the code and copies the file directly to production, or whatever. The problem is that by solving that problem quickly they are creating N problems more, where N is usually a big number.

Is there any way to remove this bad behaviour? The main one for me is to avoid these people working alone. So setting a WIP in your project that forces people to pair program (instead of using pull requests, that you should [avoid](vgaltes.com/teamwork/Avoid-pull-requests/)) seems very reasonable in this situation (and not just for this reason). The main problem you'll have with this people is knowledge sharing, so adopting XP practices is very compelling.

And don't forget that although we need to try to make people work well in our team, we have the possibility of fire people. We're not talking here about culture fit (which we can discuss if is reason enough to fire someone), but about someone who is undermining your project and your team.

![duel](/images/duel.png)