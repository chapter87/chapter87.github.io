---
title: 'A doorbell camera on a photo frame with no app store'
description: 'Our wall calendar frame is a cheap Android device with no Google Play, so it cannot run the Ring app. I made it a live doorbell monitor anyway, without installing anything on the frame at all, and the bug that made the video look frozen turned out to be one wrong argument in a buffer search.'
pubDate: 'Aug 25 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

We have a digital calendar frame on the kitchen wall. It's one of those cheap Android
gadgets that shows the family diary and a photo slideshow, and mostly it just sits
there being pleasant. I wanted it to earn its keep: when someone rings the Ring
doorbell at the front door, the frame should drop the calendar, go fullscreen on the
live camera, and then quietly slide back to the diary a minute later. Simple idea.
It was not simple, and the reason why is the good part.

## The wall I hit first

The frame can't install the Ring app. It runs Android, but with no Google Play
Services on it at all, no Play Store, nothing Google. Ring's app expects all of that,
so the obvious route was dead on arrival. The same missing piece also kills any hope
of answering the door *from* the frame, so I gave up on two-way talk before I started.
It's a screen on a wall, not a handset.

So I decided to install nothing on the frame whatsoever. It already has everything it
needs, which is a built-in WebView — a bare browser view. If I can serve the camera as
an ordinary web page on my own network, the frame can just point its browser at it. All
the real work happens somewhere else, on the Raspberry Pi that already runs half the house.

## Getting the video off the doorbell

Ring has no official public API. There's a well-known community library, ring-client-api,
that talks to Ring's own endpoints using a refresh token you generate once. My doorbell is
a battery model, and battery Rings have an annoying quirk: they flat-out refuse to take a
snapshot while they're streaming ("this camera is unable to capture snapshots while
streaming"). So the still-image approach dies the instant you go live. Fine. I pulled the
live stream itself instead. The library hands Ring's video to ffmpeg, and I have ffmpeg turn
it into a rolling stream of JPEG frames, which any browser can show with a single image tag
and no plugins.

The Pi runs a small bridge that watches the doorbell. When it sees a press it reaches over to
the frame and, in one command, wakes the screen and launches the frame's browser fullscreen on
the camera page; a timer later, it sends the frame back to the calendar app. The frame happens
to run a developer build of Android that left its debug bridge open on the network, so the Pi
can drive it without anyone touching the screen. Convenient for me on my own LAN. I would not
want that on anything facing the internet, and I've noted it as something to lock down.

## Three bugs, three lessons

**The mush.** My first working version stretched the camera to fill the whole frame. It looked
terrible, a smear of soft blocks. I assumed my scaling maths was wrong. It wasn't. Ring's live
stream is genuinely small, roughly 480 pixels square, and blowing that up to a big panel just
magnifies every block in it. The fix was the opposite of instinct: stop upscaling. Show the video
at its own size, centred on black. Smaller, yes, but sharp. A low-resolution picture shown
honestly beats a big blurry one every time.

**The late ring.** My first trigger leaned on Ring's push notification reaching the library. It
worked, but it lagged, sometimes ten seconds between the press and the frame reacting, which is
worthless for a doorbell. So I stopped waiting to be told and started asking. The bridge now polls
Ring's list of active "dings" once a second and fires the flip the moment a press appears. I also
collapsed the wake-and-launch into a single chained command instead of four separate round-trips
to the frame. Press-to-fullscreen dropped to about a second.

**The slideshow.** This one nearly beat me. I kept seeing the video "block" and freeze, no real
movement, even though every log said it was streaming fine. I almost talked myself into it being
my imagination. That would have been the wrong call. When I actually measured the frames arriving
at the browser, it was one or two a second. A slideshow, not video. The cause was small and
precise and a bit embarrassing: my code splits the MJPEG stream into frames by finding the JPEG
start marker, the bytes `FF D8`. I'd written that search to look for the *text* "ffd8" instead of
those actual bytes. So it found a real frame boundary almost never, threw most of the stream in the
bin, and delivered a trickle. One wrong argument to a buffer search.

Two changes fixed it. First, search for the real marker bytes, not the text. And second, which is
what made it properly smooth, I stopped writing each frame to a file and re-reading it — that was
racing itself and dropping frames — and instead pushed every finished frame straight through memory
to the browser the instant it was complete. Measured afterwards: sixteen frames a second, every one
of them different. That's video. The lesson I keep having to relearn is this: when a person tells you
"it's not smooth" and your logs say "it's fine," believe the person and go measure the thing their
eyes can actually see.

## Where I stopped

The frame now shows the live doorbell, sharp and moving, fullscreen the second someone rings, and it
slides back to the family calendar a minute later. If the screen was off, the ring wakes it. What it
still can't do is talk back, because no Google Play means no Ring app means no intercom, and I decided
chasing that any further wasn't worth digging deeper into the device. A viewer, not a handset. Good enough.

The part I like: I turned a locked-down photo frame into a doorbell monitor without installing a single
thing on it. Everything difficult runs on a Pi that was already sitting there, and the frame just points
a browser at a moving picture. No app on the wall, no cloud login living in the kitchen, no Google. A
constraint like "you cannot install anything here" sounds like a blocker, but it quietly forced a cleaner
design than I'd have bothered writing if the easy road had been open.

*Tech: Ring battery doorbell, ring-client-api (Node), ffmpeg turning the live stream into MJPEG, a
Raspberry Pi bridge, an Android WebView driven over adb, and the frame's own browser doing the display.*
