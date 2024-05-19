+++
title = "Minimalistic web design"
date = 2024-05-16

[taxonomies]
tags = ["rant"]

[extra]
toc = false
+++

Recently a blog post called [JavaScript Bloat in 2024](https://tonsky.me/blog/js-bloat/) made the rounds, essentially putting up a wall of shame for the most bloated websites, with Slack taking the crown shipping a whopping 55MB of bloat in Javascript alone.
If this is what modern web development best practices have ended up in, then it is not surprising that people think they need a CDN, load balancing and a Kubernetes cluster just to deploy their simple blog, which ideally would just amount to serving a bunch of static files.

Personally, I run this blog on a single virtual server and I often get asked why I don't use something like Github Pages, after all I could get the benefits of a global CDN for free.
First of all I just like to self-host all my own stuff, because it gives me total control over everything.
But realistically, there really is no advantage to using a CDN for simple websites like a blog.

Following some basic web hygiene, any website will still load reasonably fast even when accessed on a spotty connection from the other side of the globe.
In fact, I dare you to press `F12` and enable simulated throttling in the network settings, then reload this website. It will probably still load faster than most other websites over a stable fiber-optic connection.

The benefits are not limited to the client side only. The server side will appreciate some minimalism just as much.
When simulating a heavy load with [vegeta](https://github.com/tsenart/vegeta), I can go well over 10000 accesses per second until I even start seeing barely any latency in response time, no load-balancing needed.

The very website you are viewing right now fits in less than **15kB**, including the entire CSS and SVG favicon data and the 0 bytes of shipped Javascript code.
And there is no need at all to make a compromise on design, features or responsiveness.
All the really useful features are supported natively in CSS anyway, for example [prefers-color-scheme](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme) can be used to follow the system dark mode without the need for any Javascript.
Heck, CSS even supports fancy [animations](https://developer.mozilla.org/en-US/docs/Web/CSS/animation), if you want to go really wild.

# Use system fonts

Not using the system fonts is probably the biggest offender for web bloat even among blogs that are usually otherwise relatively minimal.
Let's stop pretending for one second like as if choosing a random quirky font on [Google Fonts](https://fonts.google.com/) would give a personal touch to your website.
All you are doing is increasing the load time of your website for what is basically ruining the experience of the user, who probably just prefers to use their system font on all websites.

It's ironic, how Apple users are known for being very picky when it comes to choosing non-native apps on MacOS:
Using an inconsistent UI on the desktop is a war crime, but when you do it on the web, then you are suddenly adding character to your design.

There is also no sane way to [load fonts](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display) without annoying the user. If you use `font-display: block`, then you keep the user waiting even though the text is technically already ready.
Even worse, if you use the recommended `font-display: swap`, then the user will see a very visible flicker once the font swaps in.
Granted, this usually only happens once on the first uncached page-load, but it is one of those things you can't unsee, when you have seen it once.

To top it all off, many websites delegate their fonts to `fonts.google.com`, making sure that everyone pays their fair share of data collection taxes to Big Brother.

All this is nonsense, when you can just use the installed system fonts instead. The [system font stack](https://systemfontstack.com/) couldn't be more easy to use and you automatically get the beautiful typefaces of the respective OS.
Apple users get to use their beloved [San Francisco font](https://developer.apple.com/fonts/), Android users get to use their [Roboto](https://m3.material.io/styles/typography/fonts) and everyone is happy (well, minus the Linux user who is still reading through `man fonts-conf(5)` to figure out how to enable font stem darkening).

# Avoid analytics

Similar to cookies, I try to avoid any form of analytics and tracking as much as possible.
If you really think the knowledge that 3 people read your blog is going to motivate you to write more, then go right ahead, but in that case you might as well just use the access log of your Nginx installation and a [suitable frontend](https://goaccess.io/) towards the same effect.
The really important analytics for at least all static content sites can be done entirely server-side. Adding some Google Analytics or [Firebase](https://firebase.google.com/) garbage on the client-side is so utterly backwards, when you serve an otherwise static website and even for many dynamic web pages the usage is questionable at best.

# Conclusion

While it is easy to take [minimalism too far](https://motherfuckingwebsite.com/), I think the current state of the web is going a little too far in the opposite direction.
If reading your website involves clicking away three different popups, while in the process eating up my monthly mobile data, then I am probably just not going to bother reading it.
And unless you are a Google-sized company, you should double-question if you really need that CDN and overengineered Kubernetes setup just to serve your static files.
