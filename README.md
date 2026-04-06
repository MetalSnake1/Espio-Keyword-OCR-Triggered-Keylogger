# Espio-Keyword-OCR-Triggered-Keylogger
Context-aware keylogger/credential harvester - passive OCR triggers active keystroke capture via indirect syscalls.


## Demo Video

[![Watch the video](https://img.youtube.com/vi/AR7NzGZQFto/maxresdefault.jpg)](https://youtu.be/AR7NzGZQFto)

Click the thumbnail above to watch the demo on YouTube.


----------------------------------------------

## Screen Duplication
There are several native API options for Windows screen duplication. The primary ones that I am aware of are DXGI Desktop Duplication and GDI. For this program I decided to use GDI. It's compatible with Windows all the way back to Windows XP. The API is simpler than DXGI. GDI also has a low dependency footprint - it's just gdi32.dll, which is already loaded in every Windows process. However, there are some cons. It is slower than DXGI, especially for full-screen capture. It still works fine as you saw in the video, but on slower machines, or maybe some virtual machines, it might lag behind. I still need to do further testing to get any accurate measurements though. GDI also doesn't capture hardware-accelerated content well. Some DirectX/OpenGL apps might show up as black rectangles. This API is also CPU-bound. So, I really used GDI out of compatibility, stealth, and ease of use.

DXGI captures frames directly from the GPU, which means lower CPU usage. It can capture hardware-accelerated content that GDI misses. It's what OBS, Discord screenshare, and most modern capture tools use. There are some drawbacks though. It is only available on Windows 8+. The API is also more complex. DXGI duplication requires the process to hold a reference to the GPU output adapter, which means if EDR is monitoring Direct3D device creation from non-graphical processes, that's unusual behavior. You don't want a background process or injected DLL suddenly creating a Direct3D 11 device and then calling AcquireNextFrame. That is more of a distinctive behavioral signature than what GDI does. GDI calls GetDC/BitBlt - hundreds of legitimate applications do that constantly for rendering UI elements.

Now, there is a problem with both DXGI and GDI. Some applications use an API function called SetWindowDisplayAffinity. That blocks GDI BitBlt and DXGI Desktop Duplication. You would get a black rectangle if you were to look at the captured frame. So, the thought I've had is to create a fallback capture method which can bypass SetWindowDisplayAffinity. That is a rabbit hole though, and not one I've really gone down yet. I definitely want to look into that at some point.

## OCR Engine
I'd like to briefly talk about the OCR engine. Windows has a built-in OCR engine called Windows.Media.Ocr.OcrEngine - you feed it a bitmap, and it returns recognized text. It is native to Windows, so every Windows computer from Windows 8.1 onward can utilize the OCR engine. (Thanks Microsoft! Please implement a passive OCR feature for Windows 12!) I like the built-in Windows OCR engine because it "blends in better" - there are also built-in apps like the Snipping Tool, Microsoft Office apps (OneNote in particular), and Windows Recall/Copilot+ that use it - anything that would need to extract text. It's also nice that there are no additional dependencies. The Windows OCR engine automatically handles whatever language the target has configured. Tesseract, another OCR engine, would require you to bundle language data files for each language.

## Why Not URL-Triggered Keylogging?
Now I would like to briefly address something that I'm sure some of you have thought of. Why not just get rid of the OCR engine and use URL-triggered keylogging? Which can be good, but sometimes you don't have any solid recon information. You might be targeting non-browser apps, dynamic URLs, and scope changes. So, that's why an OCR credential harvest might be the better tool for the job.

## Keylogging via Syscalls
I'd like to now get into how I actually did the keylogging. This is the first time I've worked with syscalls, so it was pretty interesting for me.

Most keyloggers use an API function called GetAsyncKeyState for logging keystrokes - the problem is that a lot of EDRs flag it. There are some workarounds though. GetAsyncKeyState doesn't actually check the key state at a user-mode level. It's a wrapper - what it's really doing is calling NtUserGetAsyncKeyState in win32u.dll, which then executes a syscall into the kernel. The kernel is where the actual key state information lives. So GetAsyncKeyState is just a user-mode function which acts as the front door to get that syscall into the kernel.

To get around that front door, the first thing you have to do is find what the wrapper function is actually calling to.

## Why EDRs Hook at a High Level
I'd like to take a quick detour and explain one reason that most EDR vendors usually hook functions at a high level. In this case, for GetAsyncKeyState which is in user32.dll, it's because those functions have the most stable API - it doesn't change between Windows updates, so hooks don't break. If an EDR were to hook into win32u.dll, they would have to constantly deal with compatibility, because Microsoft can and will change the internals across builds. user32.dll is just more stable. However, some EDR vendors do hook at a win32u.dll level, so this isn't a total workaround.

## What Is a PEB?
I also need to quickly explain what a PEB is. PEB stands for Process Environment Block - it's a user-mode Windows structure that holds important metadata about a running process, such as loaded DLLs. Every process has its own PEB.

Finding and Executing the Syscall
Now that you understand contextually and conceptually where I am coming from, you will understand why I needed to make a syscall. First, I needed to find where win32u.dll is actually loaded - I used PEB walking to do so. After doing that, I did something called PE Export Parsing - this tells you where in win32u.dll NtUserGetAsyncKeyState is actually loaded. After that you will finally have a syscall number which can be used to call NtUserGetAsyncKeyState and get around that pesky front door some EDRs hook.


## Indirect Syscalls
Once you finally have a syscall number, you have a problem - the return address in the call stack points to your RWX memory allocation, not a Microsoft DLL, and modern EDR does stack analysis. So, instead of executing the syscall instruction in your memory, you find a syscall; ret gadget (bytes 0F 05 C3) that is already inside win32u.dll - then you jump to it. Now the return address of the syscall points to a legitimate Microsoft module (win32u.dll) instead of your own RWX memory allocation.


## C2 Communication
For C2 communication, Espio sends data to Discord webhooks using WinHTTP over HTTPS. The traffic goes out to discord.com on port 443, which looks like normal Discord traffic at the network level. However, endpoint detection will see that an unknown process is talking to discord.com - that is a red flag. Also, some network monitoring tools will see a process creating discord.com traffic from a process that is not Discord.exe. Discord actively scans for webhook abuse. If a webhook is reported, or if Discord's automated system detects unusual patterns, they kill the webhook URL - effectively making Espio useless. So, Discord is definitely not the best or even a good C2 solution in my opinion, and honestly creating a good C2 server/system is something that I am looking forward to doing. I definitely need to further learn about good offensive networking strategies.

## Some Problems

------------------------------------------

RWX Memory Allocation

My indirect syscall stub is 21 bytes allocated with VirtualAlloc, using PAGE_EXECUTE_READWRITE. That is well known by EDR heuristics. Legitimate applications almost never need memory that is simultaneously writable and executable. A lot of popular EDR products monitor for private RWX (read, write, execute) allocations. Some even flag on the VirtualAlloc call itself.

One workaround I was thinking of is that if I could make it so Espio masquerades or injects into a JIT process, its own RWX memory allocations blend in with the process's expected behavior. A JIT process is any process that compiles code at runtime rather than ahead of time. A JIT process repeatedly allocates executable memory during normal operations because it is compiling code on the fly as it is needed.

-------------------------------------------------

Aggressive Polling

Here's another major point of failure. The polling in this program is very, very aggressive. It polls 80+ keys in around ~1ms, which means there are around 80,000 syscalls per second during active mode. That's unusual behavior, lol. Additionally, CPU usage spikes during the active window, which gives way to another detectable pattern.

-------------------------------------------------

Passive-to-Active Mode Latency

For the last major point of failure - it's about optimization. This mainly affects slower computers and virtual machines. I haven't personally experienced it on my main desktop. However, there will always be a latency gap during the transition from passive to active mode. This is unavoidable. When a keyword triggers active mode, Espio doesn't instantly start logging keystrokes. It has to do several things first:

First, when OCR detects a trigger word, it has to send that screenshot to Discord, but the "image" is just a raw bitmap. Discord doesn't accept those, so it has to be JPEG-encoded. Next, Espio's mode flag is set to active mode, then the JPEG is sent to Discord, polling begins, and the buffer is exfiltrated to Discord after a set amount of time.

Basically, because Espio is single-threaded, the keylogger can't start polling until the Discord upload is complete and the main loop cycles back to active mode. The reason it has to cycle back to active mode is because when the JPEG is sent to Discord, the loop is still technically in passive mode. One thread can only do either passive or active mode work - the thread has to finish its current work, loop back, read the set flag, and then act on it.

## Where This Is Going
When the Windows OS is utilizing Copilot+ / Recall, it is already doing periodic screen capture and OCR, because Microsoft wants you to be able to search or "recall" what you were doing. So, based on speculation alone, I believe that Microsoft might be working on releasing a more consistent, passive OCR feature for the Windows OS in the near future. This would make the passive mode of Espio harder to detect, because it would begin to look more like a Windows feature rather than malware. However, Microsoft has restricted Recall in the sense that you have to opt in to use it. I've also read that Microsoft is considering and/or is working on reworking Recall. So, Recall could potentially be removed in the near future - I have no idea. However, I don't think the overall trend towards OS-level machine learning/screen understanding is reversing.

This leads me to be interested in developing context-aware malware, using Espio as a fundamental component of later research. I have been thinking about ways to implement persistence for Espio, and I would like to implement a sort of watchdog that basically looks for Espio, sees if it stops, and then if it does stop, the watchdog will respawn Espio again. The watchdog would be a Windows process that utilizes some sort of obfuscation - I haven't really read too much into this, but from my understanding, it should be possible. There are a few other things I would like to implement, but it's mostly theoretical - I just want to say that I plan on evolving Espio.

## Near-Future Features
For the near future, I want to implement these features:

- Quieter polling with reduced key sets - I've got to get those 80,000+ syscalls per second during active mode down. I need to reduce which keys are actually polled, because usually only about 50-60 keys are actually used in passwords/usernames.
- Clipboard and cookie exfiltration.
- Get rid of Discord as the C2 server. I'd like to include bidirectional communication, so I can include keyword list updates, configurable polling parameters, a kill switch, and tasking.
A display affinity bypass.
