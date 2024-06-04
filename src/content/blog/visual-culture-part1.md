---
pubDatetime: 2019-06-28T16:22:00Z
title: Visual Cultures Part 1 - XFDs
featured: false
draft: false
tags:
  - xfd
  - visual
  - metrics
  - ci
description:
  Using eXtreme Feedback Devices for build monitoring - part 1.
---

My team (when I say team, I actually mean 14 individual teams at the moment) is using big visible dashboards since they were not **a thing**. 
In fact, not only dashboards, but our entire culture is very **visual**. We like things to be **transparent** and easily **accessible** with a simple look. 
This applies to **everything**: sprint/iteration progress, code quality, system test environments health, release readiness (I know, but not all of us are doing fully-automated-kubernetees-driven-continuous-delivery-with-10-deploys-per-day), build stability and so forth. 
Even if everything is accessible through a web interface, we prefer to also have a physical or dashboard-visual stuff. It’s instant and meaningful.

One of the first things we’ve introduced was an **eXtreme Feedback Device** (XFD). There are many geeks within the team. 
What better way to express your geeky skills than DIY techy projects? And an XFD seemed like a very good candidate. 
For those unfamiliar with what an XFD is a short definition would be an _instant and visual way you can monitor a Continuous Integration build health_.

And so, we’ve built one :)

![XFD Version 1](@assets/images/xfd1.png "XFD Version 1")


## XFD 1.0
The first version of the XFD was built using:
•	an Arduino board
•	an RGB LED
•	a Java agent that monitored the CI server’s builds and was sending bytes according to the health on a serial port to the Arduino board
•	some Arduino code that was interpreting the bytes from the Java agent and setting the LED lights accordingly

And it worked very well. Build **failed**! Bang!!! XFD was turning 🔴. Build **in progress**! Bang! XFD was pulsing 🔵. System test environment was **unhealthy**! Bang! XFD was turning **yellow**. 
You get the point. We now had instant visual feedback for things very important for the team.

The Java agent was configurable so you could accommodate multiple CI builds or different team events (like daily stand-ups) based on specific time frames.

Over time, as the number of teams increased, the need for **diversity** increased. This is why we’ve created different flavors of the XFD. 
They were working in the same way but had different aesthetics.

![More minimalistic XFD](@assets/images/xfd2.png "More minimalistic XFD")

![Christmas version](@assets/images/xfd3.png "Chistmas version")


## XFD 2.0

We’ve used the 1.0 solution for almost 6 years. During an office move last year, we’ve noticed that everything was fine as long as you leave the XFD in one place. 
Once you need to move it around, it was a bit difficult for the 2018 geeks to cope with having so many wires (power-in, power-out, USB) and bulky designs.

Welcome XFD 2.0! We didn’t actually `build` this one per se. We've bought Xiaomi Yeelight RGB light bulbs. 
Quite cheap, portable and API driven :) Very geek friendly. We’ve re-written the Java agent from scratch in order to communicate with the bulb. 
And everything was working like before, but in a cleaner way…

… until the need for **diversity** stroke again. After all, what each team had was just a light bulb. 

So, we’ve launched a **competition** between teams. 

Who can create the best looking XFD (while still being powered by a light bulb)? I must say that the results are impressive. 
We now have 12 really cool concepts that in the same time capture the uniqueness of each team.
All pictures below.

![Simpsons](@assets/images/simpsons.jpg "Simpsons")
![Jedi Smurfs](@assets/images/jedi.jpg "Jedi Smurfs")
![Fight Club](@assets/images/fightclub.jpg "Fight Club")
![Transformers](@assets/images/transformers.jpg "Transformers")
![Black Sheep](@assets/images/blacksheep.jpg "Black Sheep")
![Wolfpack](@assets/images/wolfpack.jpg "Wolfpack")
![X-Men](@assets/images/xmen.jpg "X-Men")
![Florence](@assets/images/florence.jpg "Florence")
![Monsters](@assets/images/monsters.jpg "Monsters")
![Incredibles](@assets/images/incredibles.jpg "Incredibles")
![Lucky 13](@assets/images/lucky13.jpg "Lucky 13")

## XFDs are so early 00’s - Do they really work?

I **strongly** believe they do. But they **don’t** “just work”. **You must create a specific mindset**. Cultures do not emerge because things are there - that’s it; people need to behave in a certain way. 
They emerge if they are part of a **bigger picture**. From a mindset that considers that **visual items are important**. And that visual ways to show stuff are key for transparency and instant feedback.

## The code for XFD 2.0
You can also start your own XFD in 2 minutes (after you buy a Xiaomi Yeelight RGB LED). The code is on GitHub: [https://github.com/ludovicianul/yeextreme](https://github.com/ludovicianul/yeextreme)

**Be visual!**

