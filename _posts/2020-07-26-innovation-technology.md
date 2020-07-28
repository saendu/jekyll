---
layout: post
title:  "The strange relationship between innovation & technology"
author: sandro
categories: [ DEV ]
image: assets/images/innovation-technology.jpg
---
Innovation is driven by technology. Or is it the other way around? Actually, it is even more complicated than that. The relationship between technology and innovation is somewhat 'special'. 

**TL;TR;** Build your products to be ready to rebuild them.
<div style="width:100%;height:0;padding-bottom:77%;position:relative;"><iframe src="https://giphy.com/embed/91fEJqgdsnu4E" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div>

### Building MVPs
When we build MVPs or prototypes, we need speed. Lots of speed. We are agile, we change and when we are done, we know what we wanted to know. And then, what happens next? Well we probably start from scratch and build the product with a solid base, right? Yeah sure...
If every time I would have got a nickel when I heard the following statement in my past IT career:

> Let's not make things more complicated than we need to. We are anyway gonna throw this away. 

I would now been slurping my Caipirinhas somewhere nice and warm and for sure not writing blog posts about IT. 
<div style="width:100%;height:0;padding-bottom:56%;position:relative;"><iframe src="https://giphy.com/embed/5xtDarqlsEW6F7F14Fq" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div>
Or would I?

Representative studies (actually just my experience) showed that 9/10 MVPs or prototypes will survive the guillotine and proceed on. Resulting us having products on a wobbling base. Nobody wants that except your competitors. And this makes us feel like:
<div style="width:100%;height:0;padding-bottom:56%;position:relative;"><iframe src="https://giphy.com/embed/l1KVaj5UcbHwrBMqI" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div>
But they have these kinds of products probably too, so don't get too sad. 

> Ok, I attracted you in with some Giphys, now let's get to the facts. 

### Hitting the sweet spot
So, when is the right moment to say: now is the time to work on that refactoring. And lose a little bit of that speed for a good investment into the future - into a stable future. Unfortunately, this is the million-dollar-question and I can't give you the perfect answer to that (see Caipirinhas-reason above) but I try to give you some guidance:

- **How large is your technical depth?**
Ask yourself how much time you would need to rebuild the whole thing. Make the calculation with the same and with a new team. 
- **Will this product have good chances to survive the next six month?** If it won't be killed in the next six month you probably have a reason to believe why it will survive this fairly long period. In six-month time a lot of things can change. For example, the product can substantially grow in complexity if you work on it or the opposite – nothing happens, and important members of your team leave. And further development will be much harder. In the latter case you are now stuck with your technical depth. 
- **What happens if it really works and gets big?** If this question somehow frightens you then be honest with yourself and ask yourself if it is caused with you procrastinating an important 'refactoring' question or something similar. 

### The technology choice
So, let us focus on another topic: **Technology**. Should we always use the most recent technologies out there? Well, if we call ourselves 'innovative' we probably should, right? This sounds like a no-brainer, but it is a tricky beast. Betting on the latest technology is a double-edged sword. It might save you from running on an outdated stack, but some technological choices can put you also into a corner where it will be very hard to maneuver out. Especially if the technology is very 'hipe' you might not find many others that support or even understand it. Especially if you are dealing with enterprise IT operations. They are might one or two years behind the current market which puts you in a standoff where you are unable to integrate your product anywhere but your DEV/TEST environment. 

### OAM & Dapr to the rescue
New initiatives like the [Open Application Model (OAM)]( https://oam.dev/) and [Distributed Application Runtime (Dapr)]( https://dapr.io/) might help you in the future to build technology agnostic applications and avoid the frightening vendor lock-in. I will write an article about that topic soon. But since this is a very recent development, we might have to make technological choices in a fairly early stage. So, try to talk about your stack as early as possible and start with the most promising technology from the beginning. 

### Some key take-aways
If Facebook still would be a thing, then the relationship status of technology & innovation would be 'it is complicated'. And this time it is the truth. Here are some final advice I can give you to not fall into the same traps I did in my past. 
- **Choose your technology wisely:** Don't just use 'something'. This might sound like nobody would ever do that, but I saw that more often than I expected. 
- **Choose a modular architecture:** The term sometimes sounds a bit frightening to developers but modular architectures are not just something for very large enterprise architectures. They actually make plenty of sense also for smaller applications, especially when we expect things to change in the future. If you have a modular architecture, you are prepared to swap a module out without the pain of a full-blown refactoring. 
- **Use OAM and Dapr:** Yes, these schemes are pretty new on the horizon. But in the future, you should consider using something like OAM and Dapr to avoid vendor lock-ins and non-interchangeability. Especially if you build your products in the cloud. I will write an article about that very soon.

And for a final thought: 
> Be ensured that you will never do it right the first time. But this is not what counts – what does; is to be prepared for when you need to fix the things you did wrong.

. . . Until next time
