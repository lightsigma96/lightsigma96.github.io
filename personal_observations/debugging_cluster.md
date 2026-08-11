## My experience debugging valkey daily workflow failure

For past week I have been tyring to look at error caused in daily workflow in valkey, hoping to make a PR with my goal to create 2 more.

Compared to my last PR which was pretty trivial, this was some good deep dive into valkey internals involving various subsystem.

The first error was about cluster slot manual and atomic migration, where one of the test cases was expecting eventual failure when manual migration is under process.

So I first tried out both MANUAL & ATOMIC migration and it was working as expected and all tests were passing, I was not able to reproduce the failure. 

This is the **first important point** to **reproduce environment**, as I spent a lot of time on normal environment where as
the failure was in threaded + tls ubuntu environment, I later used docker for this environment.

So the tests obviously expected "OK" but instead server gave "Already exporting slot N", and so hence I tried to look for this exact string and
tried to trace the slot migration path.

Eventually I found that the job did not reach terminal state (failed/sucecess). I thought of many different possibilities, the next day saw a PR concluding the
same also stating this was due to a race condition which explains why it saw a job in non-terminal state when not expected. (well there are more details to this
and I might be wrong)

This is the **second important point** that always look at something unique in the problem, here it was threaded environment as the failure was only in that
environment and not in any other.

There was another test failure again in same environment which was about a delay, and so I again traced the path and tried to find which step was causing the delay.

Well that did not go anywhere, but then was another test failure "I/O reading error", again in same environment.

This is also a sign that there is actually something wrong and that this is not just another falkey test.

This failure was due to client connection abruptly breaking due to some reason.

My first thought was that due to threads some other thread might be closing the socket descriptor on the client while some other thread was using this and that the socket descriptor
was not locked by mutex.

Later I found that valkey/redis, actually does not use a traditional mutex pattern which is used in multithreaded system, they actually transfer ownership of job from main thread to 
child thread and uses state to define if child threads are actually busy, so the possibility of race was again not valid.

Later when I looked into **stack traces** which is again **very important** and I should I done early, I found there was an assert was causing closing server abruptly and this was a stronger
evidence, initially I tired to supress the assert but this is not a good approach instead the cause should be fixed.

It was almost a week now on these errors, that for this final step of finding cause I called AI to point the last missing piece as I had already found that the assert was due to double blocking,
but did not know what exactly was the path that led to this double block, that path was given by AI and finally I raised a PR.

The PR was not merged instead the other one raised by the contributor who brought this regression gave the same fix was merged as they had already dealt with something like this.


**I realised one thing that I was just reading code and understading architecture, and the final fix for this week long understanding was just 4 lines of code which suggests that code is just a small part of solving a problem and mostly
it is understading the architecture around the problem and finding correct invarients violated.** 


