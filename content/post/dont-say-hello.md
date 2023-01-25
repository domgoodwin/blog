---
title: "Don’t just say ‘hello’ to me"
date: 2023-01-25T19:06:09Z
draft: false
categories:
- Productivity
- Ways of working
---
I was recently in a 1-1 with my new EM and was asked what annoys me at work. The one thing that immediately popped to mind was:

> ‘I hate it when someone just messages me “Hello” or “Hi, how are you” when they message me for something on Slack’

It became immediately apparent that, without proper context, this could be taken the wrong way. So I thought I’d explain my thoughts here.

<!--more-->


It has annoyed me since the advent of instant messaging at work, but as more people now work remotely the problem has only got worse. 

At a high level, the issue is this: instant messaging is asynchronous, unlike a verbal conversation, each participant can be doing something else at the beginning of the chat and might continue doing so for the duration of the conversation.

With a phone call, communication is instantaneous, so the time for pleasantries is tiny compared to the actual discussion of a problem. When it’s via chat there’s delay, even if both parties are watching the chat, waiting to respond as soon as possible, they still take time to type and find the write emojis.

Let’s use a sequence diagram to show what I mean:

{{< figure src="/img/nohello-1.1.png" title="Bob and Alice talk in serial" caption="Bob and Alice talk in serial" >}}

This is a pretty typical conversation, even ignoring Bob’s rudeness of not asking after Alice. Alice needs to ask Bob a question, but starts the conversation as you would in-person or by phone. If we think of humans as single core processors, both are basically ‘blocked’ for the entire duration, unable to do anything else until this conversation is over.

This gets worse if you factor delays into the responses:

{{< figure src="/img/nohello-1.2.png" title="Bob and Alice talk in serial with delays" caption="Bob and Alice talk in serial with delays" >}}

Now obviously this is a worst case scenario, both parties could be doing other things whilst waiting for responses, but that would also introduce more interruptions to their work. Adding a context switch every time someone sends a message would slow down whatever work they were doing, as an example:

{{< figure src="/img/nohello-1.3.png" title="Bob and Alice talk in serial with context swtich delays" caption="Bob and Alice talk in serial with context switch delays" >}}


Here, Bob is trying to complete his work whilst also responding to Alice. Each time he responds then returns to his work. However, this means he needs to context switch and remember what he was working on before the interruption.

Now, how could this be improved:

{{< figure src="/img/nohello-2.1.png" title="Bob and Alice exchange a single question" caption="Bob and Alice exchange a single question" >}}

Alice has a question, she messages Bob the question but then go back to their work (if possible).

Bob receives the question, thinks through an answer and sends it back to Alice who is then unblocked. 

Alternatively, Bob is working on a tough problem so doesn’t see the message until 30 minutes later, he then responds and Alice who was looking at a different problem can then get her answer.

Regardless of how long a response takes, both parties spend the same time on the communication. Compare that to earlier examples, even if Bob context switched with every message, the time consumed with the same outcome (Alice has her answer) is far greater.


> ℹ️ A lot of this boils down to just sending someone [https://www.nohello.com/](https://www.nohello.com/) which explains this a lot more succinctly than I could.