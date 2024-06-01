---
title: "The case for time-tracking"
date: 2022-02-05T15:06:09Z
draft: false
tags:
- productivity
- ways-of-working
---
Time tracking has a bad reputation, it invokes images of managers watching over your shoulder or pay being docked as you punched in minutes before your shift ended. Due to these, and many more, if you suggest to someone to start tracking their time they’re likely to look at you like you’ve gone mad.

<!--more-->

I think time tracking, for your own benefit, can be a great way to be introspective about how you work and allow you a greater understanding of managing distractions, context switching and when you’re feeling productive.

## What is time tracking

The basic gist is, you start a timer when you are working on something. Whilst you could track time per ticket or some other fine grained metric, I would strongly recommend tracking categories of activities.

Ones I track are:

- **Development** - directly coding, usually “when I’m in my IDE”
- **Live support** - picking up alerts that have fired or investigating issues/incidents
- **Messaging** - slack messaging people
- **Meetings** - time spent in any meetings
    - I found it in the past useful to *tag* these “ceremonies”, “1-1s”, “other”
- **Break** - Going to grab a coffee or lunch
    - This one could be tracked by just not running a timer, but I find ending the Break timer when I return is a useful trigger to start working
- **Collaboration** - Working with someone else to either plan, pair or debug

These activities can also be expanded with tags if you want to know more detail, things like whether the activity is for client a or b can be useful if that fits your environment.

I’ve also found adding activities when you want to focus on a particular area, say “Documentation” or “Reading tech books” can help you focus energy towards a goal.

A key thing with the tracking is not being too detailed, it’s not useful to know ticket-12 took twice as long as ticket-14 in the scheme of things (even if Toggl do like to push jira integration). You’re not billing people for this time, it’s just about gaining understanding.

## How to time track

This might seem like an obvious area “set timer to start when you work on something” but there’s some nuance and tips I can share.

- If at any point you stop working on your current task for distractions, stop said timer (and set a new one if applicable)
    - A common thing might be you need to respond to someones message, ideally you try to batch these where possible and the people messaging you [don’t just say hi](https://www.nohello.com/), so maybe after 30minutes, stop the timer, start the messaging timer and respond to people
- Before you begin any activity, start the timer
    - I find this is a useful way of getting in the mindset for that task too, it forces you to mentally think “what am I working on right now” instead of just falling into work
- You can always update past timers if you forget too, don’t try to be too accurate, to the nearest 5 minutes should show you the gist of it.
- “Doesn’t this add to the context switching load?” **Yes** I believe it does, but that isn’t a problem. It should be harder for you to switch what task you’re working on to avoid (where possible) context switching and having to recognize you’re doing it, stop and start a timer, adds to the overhead, it’s way too easy to down tools on work to respond to slack.

## What time tracking is not

It’s important to call out what personal time tracking is not, and avoid some of the pitfalls.

- Time tracking should be for your own benefit, and you should be the only one who sees the data. You’re not doing this to prove to your manager at year-end reviews you keep working even when you sneeze or to compare to your team. On the other side, this shouldn’t be something a manager imposes on you and wants to see progress reports.
- Related to above, it’s important to be honest when time tracking. Everyone has bad days for productivity and just leaving “development” timer on won’t help you understand anything. This timer is just meant to be data on how you work, no opinions or judgement.

## Benefits of time tracking

I’ve personally found tracking my time, even for short 1 month stints, a great way to introspect on your working habits.

A good example is at the end of the day, when you might be feeling like it was (or wasn’t) a productive day. Being able to look back and go “oh, those 4 meetings 30minutes apart basically translated to 4hrs of meetings for my productivity”. I’ve commonly found days when I’m feeling unproductive, I can look at my time tracked for the day and understand why.

Another key aspect is seeing the effect interrupts have on you, everyone knows the bane of context switching at this point, but being able to see the switch in activity and then the lull between starting anew is clear.

I’ve also found having to think about *starting* a task, having a timer to start when I go into development mode helps mentally adjust to the task at hand.

## Tools and use and how to set them up

For actually tracking the time, I use: [toggl track](https://toggl.com/track/). Honestly, I don’t particularly like it but I haven’t found an alternative that does everything I need and don't need to install.

I’ve previous used [Timery](https://timeryapp.com/), which makes it easier to change timer and highly recommend it to never interact with toggl. I would highly recommend this when you can install applications on your own machine and for iOS if you have it.

- I have a pinned tab open in my browser with toggl, it goes grey when there isn’t one active which I find useful
    - Timery will sit in the top bar of my Mac which makes using it easier
- I then setup a widget on my phone home screen. This helps me spot any timers I’ve left running.

Within toggl, I have the activities I listed above as “Projects”. I don’t use any tags currently but I’ve found it useful in the past when working across teams or clients.

- A good rule is a project should be something important to you but broad enough you don’t need to think or decide which one to use when starting a timer. It should be obvious what the work you’re doing is.
- As an example for tags, I wanted to see which client’s live issues were taking the most time in my day.

## To round up

Time tracking can be a useful tool to gain insight into your working habits and productivity. I’d recommend only doing it when you’re gaining something from it; in the past I’ve used it for 1-3month stints when feeling unproductive or joining a new company. If you’re doing it and finding the data not useful.