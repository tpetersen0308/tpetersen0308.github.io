---
layout: post
title:      "There are worse problems to have"
date:       2018-01-15 01:10:39 +0000
permalink:  there_are_worse_problems_to_have
---


The other day, I was working on a Sinatra lab (I think it was one on user authentication), and I had one of those experiences that occur occasionally where my program kept crashing and I couldn't for the life of me figure out why. Everything seemed to be in order, but I kept getting a Sinatra error telling me that, essentially, there was no route defined for the current get request. The problem there was that it wasn't supposed to be a get request, it was suppost to be a post request from a login form. I spent what felt like at least an hour reviewing all of my views, double-checking (then triple-checking) the logic in my application controller, and scouring the login form over and over again because that's where I really felt like the problem should be. But the form was clearly attributed a "POST" method with a "/login" action. Right when I thought I must simply be crazy (or stupid), I spotted it:
```
<form action="/login" method:"POST">
```
I accidentally set the method with a f@#$ing colon. And that was it. My program worked. 

Usually, I would be totally beating myself up over this, but at this point in my programming education I've learned that these kinds of mistakes are pretty typical, and I don't make them all the time anyway. I was a little frustrated over how much time was wasted over a f@#$ing colon, but later that day I met up with some other FlatIron students and wound up recounting the experience to them. Everyone had plenty of their own such stories, and we also agreed that if such experiences comprise some of the more frustrating elements of programming, it still sounds like a pretty sweet deal. Especially because once you get everything working the way it should after all that frustration, it feels pretty damn good. 


