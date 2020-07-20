---
layout: post
title:  "The strange relationship between innovation & technology"
author: sandro
categories: [ DEV ]
image: assets/images/innovation-technology.jpg
---
Innovation is driven by technology. Or is it the other way around? Actually it is even more complicated than that. The relationship between technology and innovation is somewhat 'special'. 

> TL;TR; Build your product to be ready to rebuild it.
<iframe src="https://giphy.com/embed/91fEJqgdsnu4E" width="480" height="368" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/camera-kickstarter-possibilities-91fEJqgdsnu4E"></a></p>

### Building MVPs
When we built MVPs we need speed. Lots of speed. We are agile, we change and when we are done we know what we wanted to know. And then, what happens next? Well we probably start from scratch and build the product with a solid base, right? Yeah sure...
If every time I would have got a nickel when I heard the following statement:
> Let's not make things more complicated than we need to. We are anyway gonna throw this away. 

I would now been slurping my Caipirinhas somewhere nice and warm and for sure not writing blog posts about IT. Or would I? ;-) 

Representative studies (actually just my experience) showed that 9/10 MVPs will survive the guillotine and proceed on. Resulting us having products on a wobbling base. Nobody wants that except your competitors.
<iframe src="https://giphy.com/embed/l1KVaj5UcbHwrBMqI" width="480" height="270" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/lifetimetv-adorable-babies-l1KVaj5UcbHwrBMqI"></a></p>
But they have these kinds of products probably too, so don't get too sad. 

### Hitting the sweet spot
So when is the right moment to say: now is the time to work on that base. And lose a little bit of that speed for a good investment into the future - into a stable future. Unfortunately this is the million-dollar-question and I can't give you the perfect answer to that (see Caipirinhas-reason above) but I try to give you some guidance:

- **How large is your technical depth?**
Ask yourself how much time you would need to rebuild the whole thing. Make the calculation with the same and with a new team. 
- **Will this product have good chances to survive the next six month?** If it won't be killed in the next six month it is most likely not an MVP anymore. Also, in six month time a lot of things can change. For example it substantially grew in complexity or the opposite - nothing happened and important members of your team left. And further development will be thought - in that case you are now stuck with your technical depth. 
- **What happens if it really works?** If this question somehow frightens you then be honest with yourself and ask yourself if it is related with procrastinating an important 'refactoring' question or something similar. 


### The technology choice
So let us focus on another topic: technology. Should we always use the most recent technologies out there? Well if we call ourselves 'innovative' we probabbly should, right? Well sounds like a no-brainer but it a tricky beast. Betting on the latest technology is a double-edged sword. It might save you from running on a outdated stack but some technology choices can put you also into a corner where it is very hard to manover out. Especially if the technology is very 'hipe' you might not find many others that support or even understand it. Especially if you are dealing with enterprise IT operations. They are might one or two years behind the current market which puts you in a standoff where you are undable to integrate your product anywhere but your DEV/TEST environment. 

### OAM & Dapr to the rescue
New initiatives like the Open Application Model (OAM) and Distributed Application Runtime (Dapr) might help you in the future to build technology agnostic enterprise applications and avoid the frightening vendor lock-in. See also my article about [OAM and Dapr](#Article). But since this is a very recent development, we might have to make technology choices at a farely early stage. So, try to talk about your stack as early as possible and start with the most promising technology from the beginning. 

### Some take aways
The relationship status of technology and innovation is 'it is complicated'. And this time it is the truth. Here is some final advice I can give you to not fall into the same traps I did. 
- **Choose your technology wisely:** Don't just use 'something'. It might sound like nobody would do that, but I saw that more often than I expected. 
- **Choose a modular architecture:** Modular architectures are not just something for very large enterprise architectures. They actually make plenty of sense also for smaller applications, especially when we expect things to change in the future. If you have an architecture like that you are prepared to swap something out without the pain of a full-blown refactoring. 
- **Use OAM and Dapr:** Yes it is very new. But if you can use something like OAM and Dapr to avoid vendor lock-ins and non-interchangeability. Especially if you build your products in the cloud. I wrote an article about that to get you started [OAM and Dapr](#Article).

And for a final though; Be ensured that you will never do it right the first time. But this is not what counts - what does is, to be prepared for when you need to fix the things you did wrong. 
