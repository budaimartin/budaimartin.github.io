---
layout: post
title:  "The agony to change ringtone on your iPhone"
date:   2019-05-05 23:02:00 +0200
categories: [Today I Learned]
---

Today, after about three years I decided to change the ringtone on my iPhone (although it's being on silent all the time). And I had to find out that not much has changed since then.


# This is my ~~swamp~~ sandbox

So, you probably know that iPhone uses a sandboxed system. This means that you need special applications to access the files on your device, and still, it's limited to specific parts of the filesystem. In the past it was worse, though: you couldn't copy _anything_ to your device without using iTunes, unless you had it jailbroken.

<div style="text-align:center"><img src="/img/til-iphone-ringtone/one-does-not-set-ringtone.jpg"></div>

Of course, [when you're using iTunes](https://www.macworld.co.uk/how-to/iphone/how-set-song-iphone-ringtone-3635061/) setting a ringtone is ~~fine~~ not that bad, even it's being cumbersome and eager for updates. But what if it's not possible to use it? What if you're using Linux?

# Linux's relationship with iDevices

I'd rather leave this section empty... Really, [there's no official support](https://www.lifewire.com/how-to-use-itunes-on-linux-1999251) and [workarounds](https://www.libimobiledevice.org/) get patched all the times.

# Change ringtone without iTunes

But the good news is: [there's a way](https://www.makeuseof.com/tag/create-free-ringtones-iphone/)!

1. First, create your ringtone. It can be in any format, you don't need to convert it to M4R or AAC.
2. You now need to upload it to your device so it can be accessed through Files. Really, it can be anywhere: I used the folder of Excel documents when I copied it from my Ubuntu.
3. Launch GarageBand on your phone. If you deleted it, you can reinstall it from the AppStore.
4. Create a new "Audio Recorder" type of project: press the + sign on the right corner and swipe left until you see it then tap on it.
5. Press the third button from the left to see the tracks. The second icon from the right will change into like a loop from Hot Wheels. Tap that as well.
6. Now you should see something like a file explorer. Choose Files on the top and then "Browse items from the Files app". Do that.
7. You will now see your audio file present in a list. Drag and drop it into the track.
8. Listen to your song if you'd like to and modify it if you're into editing audio on a 4.7" screen.
9. Tap the icon on the top left corner and select "My Songs". You should see the new project you created. If you tap, hold and release it, you can rename it to something meaningful, but you should also see a Share option.
10. If you choose share, you can add it as a Ringtone. The app will warn you that it will cut it at 30 seconds (who doesn't answer the phone in 30 seconds anyways...).
11. ???
12. Profit.

So that's it. Isn't it much better than just having a button in the QuickTime player that allows you to set the ringtone? Or use any mp3 file like on any dumbphone?
