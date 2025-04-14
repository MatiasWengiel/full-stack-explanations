# Promises and JavaScript's event loop

## Introduction

Promises can be difficult to understand, when we are first learning about them.
Why do we need them? What problem are they solving? How do they work? These are all valid questions that I'll try to answer below. Before we can do that, we need to cover some basic concepts:

### Basic concepts

- **Instructions**: When we write code, we are effectively writing a set of **instructions** for the computer to follow
- **Single Threaded**: JavaScript can only use one thread in your CPU. This means that it can only execute one set of instructions at a time and can't do work in parallel\*.
  - Even when it looks like JS is doing two things at once, it's _actually_ just witching back and forth really fast!
  - *: Technically we can use something called *web workers* or *worker threads\* which allow us to use more than one CPU thread for some things, but that's way outside our scope today
- **Types of instructions**:
  - **Blocking** instructions keep the main thread busy until they finish, meaning that no other work can get done
  - **Non-blocking** instructions can be started, and allow JS can move on to the next instruction while waiting for a response.
    - A good example is making an API call - you will **never** get a response immediately, and sometimes it can take a (comparatively) long time, so it's better to do something else while you wait!
- **Event Loop**: The sequence of events that the JavaScript engine goes through while running a program. It is quite complex, but we are going to focus on two things:
  - **Call stack**: A list of instructions that JS will execute. Like a "to-do" list that JS tackles one item at a time, from top to bottom.
  - **Callback queue**: A set of instructions that are "paused", waiting for an event to happen (such as an API response or a timer running out). Once the event happens, the instructions get added back into the call stack for processing

### The problem

What if we are making a call to another server and we don’t know when (or even if) we’ll get a response? And what if the process takes a very long time?

That's where two key features of JavaScript come into play: Promises and the event loop.

Promises allow us to tell Javascript that it needs to wait for a response. For example, when we make a call to a back end API to get the user’s name, the instruction would look something like this:

function getUserData(userId) {
const userData = fetch(`${backend-url}/user/${userId}`, { method: GET })

return userData.name
}

But there’s a problem - Javascript will immediately try to read `userData.name`, even though the API hasn’t responded yet! This will cause an error, since you can’t get the `name` property of `undefined`. That’s where Promises come into play. Basically, we are telling JS “Do this, and then wait for a response before moving on to read the name field”.

That’s great, except now you are stuck waiting! Or are you? Here is a very simplified explanation of how the event loop works:

There are two main components (for our purposes, there are actually more):
The call stack - this is the list of commands that Javascript is going to execute, in order
Think of the call stack as a “to-do” list. JS always checks off one item at a time, top to bottom
The callback queue - When we need to wait for something to happen outside of our code (like a call to another server or database), it gets added to the callback queue. Imagine it as a “reminder” to look for a response.
When all the steps in the call stack get executed, JS will look at the callback queue and see if there are any responses. If there are, the next steps will be added to the call stack so the response can be processed.

The instructions we give the program are all added in the call stack, in order. But instead of having an instruction of “make this API call and wait for a response”, we can have an instruction that says “make this API call, but don’t wait for a response. Set a reminder to look for a response later on instead, and when you get a response follow these instructions”. So we “pause” the execution of a particular instruction until we get a response, but we aren’t blocking the execution of the next instruction while we wait.

Once JS finishes with all the steps in the call stack, it will go check the “reminder list” which is actually the callback queue and see if any of the responses it was waiting for arrived. If so, it adds the next steps to the call stack. If not, after checking all of the callback queue in order and if there are no new instructions added to the call stack, it checks the callback queue again from the top until all responses come in and the program can complete.

Explaining this in writing is a little complicated, but I’ll try! Let’s say that we have a few functions we want to call for our social media page called “FaceRobertson” or maybe “RobertsonBook”:
userLogin(): Requires an API call to authenticate the user. Returns an unique identifier for the user
getUserDetails(userId): Requires an API call to fetch the user’s name, unread notifications and alerts. Also gets the user’s settings
getUserPostsById(userId): Requires an API call to fetch all the posts we want to display for the user
setUserStyles(userSettings): Sets the page styles to match the user’s settings
setUserUi(userSettings): The user can choose to edit how the UI looks for them (different buttons visible/hidden, different layout, etc)
renderPage(): Actually shows the page

We could call all of these sequentially:
The user logs in
We request the details and wait to get a response
We request the user’s posts and wait to get a response
With the details we set the user styles first
Then we set the user UI
With the name, notification, alerts, styles, UI and posts we can render the page
The problem is, step 3 can’t even start until step 2 completes! So let’s say step 2 takes 10 seconds and step 3 takes 5, now we are stuck waiting 15 seconds!

Thanks to the event loop, we can do better. We can start the API call in step 2 (getUserDetails), register that we are waiting for a response and the next steps in the callback loop, then start the API call in step 3 (getUserPostsById) and do the same. This means that if step 2 takes 10 seconds and step 3 takes 5 - we only have to wait 10 seconds total! Step 3 will complete 5 seconds in and Step 2 10 seconds in, so overall we only have to wait for the slowest step to complete! Not each of them in turn.

It would look something like:
The user logs in
We request the details, with getUserDetails, and we register the fact we are waiting for a response in the callback queue.
setUserStyles and setUserUI need this data - so they get added to the callback queue too
getUserPostsById doesn’t need getUserDetails to complete, so we can make that call now, without waiting any further. Since it’s an API call, we also register that we are waiting for a response in the callback queue
Since we have nothing to render, we’ll code things so that we won’t render anything yet - we’ll just queue up the renderPage() function as well

OK! That’s the end of the call stack, so JS will look at the callback queue. Let’s say that getUserDetails returned a response, but getUserPostsById did not.
Since getUserDetails completed, now setUserStyles and setUserUi get added to the call stack and executed
Neither of them require waiting, so they complete - giving us enough information to set a page layout for the user
That was the end of the call stack so…
Back to the callback queue! getUserPostsById hasn’t completed yet, but with the user styles and ui we can actually do something for the user!
We add renderPage() to the call stack with the right styles and UI, as well as a “loading” default content
This is important, as it gives the user the feeling that things are happening!
Since the page isn’t fully rendered, we register another call of renderPage() in the callback queue
Then finally getUserPostsById returns some data! We can use that data to add renderPage with the right details to the call stack - so the full page is rendered.

Lastly, what’s the difference between Promises and async/await? As far as how the code executes, nothing! asnyc/await is a more readable way of writing Promises, but they do the same thing.

I hope this makes sense! It’s a bit of a tricky concept - so if you are still lost let me know and I’ll do my best to help.
