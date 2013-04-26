
### Who are we

I'm Sam Goldstein & I'm Ben Weintraub.  We both work at New Relic. For those of you who aren't familiar, New Relic is is an application performance monitoring service that can gives visibility into how your Ruby applications are performing in production.  We work specifically on the Ruby agent which you may know as the newrelic_rpm gem.  This is the code that runs in your app and gathers performance metrics which are sent back to our severs.

### What is a Performance Kata?

Kata (型 or 形 literally: "form"?) is a Japanese word describing detailed choreographed patterns of movements practised either solo or in pairs.  Kata originally were teaching and training methods by which successful combat techniques were preserved and passed on.

Dave Thomas coined the term Code Kata to describe a coding problem that you can solve over and over again to practice your programing and hone your problem solving reflexes.

At New Relic we've been applying this concept to performance.  Performance is an important feature of any software, and when you solve performace problems you realize that you end up solving similar problems over and over again, in different projects and contexts.  Practicing your ability to recognize the shape of these common problems helps you become a better programmer and deliver better software.

If you want to try this on your own you should check out http://newrelic-ruby-kata.herokuapp.com/ which is an app you can easily deploy for free to heroku which contains a set of performance puzzles and instructions to guide you through them.

Today though, we're going to go through another set of performance problems that we've hit on a Rails application we have been developing. We'll show you how we identified these problems using New Relic and how we fixed them.

### HANDOFF TO SAM

### Stealth Stars Inc.

Like we said at the beginning of this talk, Ben and I work on the Ruby agent team at New Relic.  But in our spare time we've been bootstrapping a secret intellegence contracting organization called "Stealth Stars Inc."  We thought it would be really cool to run a global network of secret agents.  As we've been growing this business we've been writing software to help us run it, but we've hit a few performance problems along the way.  We're going to walk through three of these performance problems and show you how we identified and fixed them in our codebase.

### Kata 1 - The Big Loop

For our first Kata we're going to walk through one of the first performance problems we hit with our Stealth Stars application.  At Stealth Stars business has been booming and we've been hiring a lot of new operatives and taking on a lot of new missions.

As our network of operatives grew we realized we were having trouble keeping track of which operatives were assigned to which missions.  To solve this problem we built out a Missions overview page which shows us which secret agents are assigned to which missions.

Here's what that looks like:

[Missions](http://localhost:8080/missions)

As you can see we have our operatives' code names on the left, and next to that the mission which he or she is assigned to.  You can probably imagine how we implemented this.  We have a Mission model and and Operative model.  A Mission has many Operatives, and an Operative belongs to the Mission they're assigned to.  Pretty straightforward, and Rails makes it dead simple to set up these types of relationships between ActiveRecord models.

This page was really helpful to helping us track who was on what mission, but we noticed that as we added more missions and operatives the page kept getting slower and slower, so we opened up New Relic to see if we could figure out what was going on.

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448?tw[end]=1366952996&tw[start]=1366950982

This is the main overview page for the whole Stealth Stars application.  This is useful for when you want to keep an eye over how you're whole app is performing, but for this problem we wanted to dive into a specific transaction - the Missions index page.  To do this we jumped in to "Web Transactions" tab a selected the transaction we're interested in from the list.

Go to:
https://rpm.newrelic.com/accounts/319532/applications/2107448/transactions?tw[end]=1366952996&tw[start]=1366950982#id=245225129

Looking at this page you can see that the response time for this action is around XXX ms, which is pretty slow, considering that we're only showing about 1000 missions.  From the breakdown graph we can see that we're spending a lot of time rendering the index template and that we're spending a lot of time in Mission#find.

To really dive into what was causing this slowness we dove into a specific Transaction Trace.  When New Relic notices a slow request it will capture a Transaction Trace which shows a detailed sequence of the events used to render that request.

Here's a transaction trace for one request to the Missions overview page.
Go to:
https://rpm.newrelic.com/accounts/319532/applications/2107448/transactions#id=245225129&app_trace_id=968699940

##### HANDOFF TO BEN

Looking at this transaction trace, we can see a chronological view of the major events that happened during this particular request, and how long each one took. Starting from the top, we immediately see a big chunk of time spent in our index template. More specifically, this red row nested under the template rendering indicates that we've got 1000 calls to Operative#find_by_sql originating from our template. Let's take a look at the code and see if we can fix it.

(Open app/views/missions/index.html.erb)

Here's our template for the operatives index page - you can see we're looping over the @operatives instance variable assigned by the controller, and then hitting the mission relation on each operative to load the mission assigned to that operative. We know based on our transaction trace that each one of these calls to Operative#mission is resulting in a separate database query - a classic N+1 query problem. What we'd like to do is convince ActiveRecord to eagerly load the missions associated with each operative using a smaller number of queries, so let's look at how we're loading operatives in the associated controller action.

(Open app/controllers/missions_controller.rb)

The easiest way to get ActiveRecord to eagerly load the missions assigned to each operative is to chain in an includes(:operatives) call here in our query, so let's do that. Here's what it looked like when we deployed this change to our Stealth Stars app in production:

Go to:
https://rpm.newrelic.com/accounts/319532/applications/2107448/transactions?tw[end]=1366955870&tw[start]=1366952559#id=245225129

And for comparison, here's an example transaction trace showing the improvement we made:

Go to:
https://rpm.newrelic.com/accounts/319532/applications/2107448/transactions#id=245225129&app_trace_id=968719245

You can see that our 1000 calls to Operative#find_by_sql are gone, replaced by a single SQL query, and our overall response time is much improved. We're ready to scale up to thousands more missions and operatives!

### Kata 2 - The Lazy Load

After we fixed our N+1 query problem we were able to get back to focusing on running our secret new intellegence network.  Our agents were running a lot of missions and pretty soon we realized that we had collected a lot of confidentional information.  We needed a good way to store and catalog this information so we got to work building an encrypted  

Around the time that we fixed the issue with our missions index page, we realized that we also had a need to enable our operatives to securely store top secret documents that they were gathering in the field. 

We got to work creating an encrypted document storage system.  We spent a little time talking with our security analysts about how we wanted our secure storage system to work, and we came up with a few requirements.

We wanted to make sure that the contents of our documents was always encrypted in the database.  This way if someone gained direct access to the database or a copy of it the data wouldn't be accessable.  We also wanted to make sure that it was easy to interact with the documents from our Rails application code which had access to the decryption key.

We wanted to avoid sprinkling encryption and decryption code all over the code base, in all the places that we needed to read or write a document, so we decided to take advantage of ActiveRecord::Base's life cycle hooks, so we could trigger encryption everytime we saved to the database, and trigger decryption after we had read a record after the database.

You can see here our TopSecretDoc class has a before_save hook which encrypts the body of the document before it's written to the database.  We've also added an after_find hook, so the body will automatically be decrypted after we read it from the database.  We also took advantage of ActiveSupport's built in MessageEncryptor class which provides a convenient wrapper around OpenSSL's various encryption algorithms.  We also run the document through the encryption algorithm multiple times to make it more expensive to crack the decryption through brute-force password guessing techniques.

This is super convenient since the rest of our code doesn't have to be concerned with the encryption logic, we just interact with the TopSecretDoc's body attribute which gets automatically gets stored as encrypted_body in the database.

Here's what the system looks like in the browser:

http://localhost:8080/top_secret_docs

We have a list of all our encrypted docs here, and when you click on a particular one, you can see both the encrypted and decrypted versions of the document body. 


After we'd launched the first version of Top Secret Docs and loaded all of our documents into it we notice that the Secret Docs index page was loading pretty slowly.  All our secret agents are paid hourly, so it's really important to us that they not spend their time waiting for pages in our application to load and feeling frustrated.

To solve this problem we ended up needing to dive into the details of how our application code was executing.  To do this we opened up the Thread Profiler tab in New Relic's UI.

New Relic's thread profiler is what's called a sampling profiler.  A sampling profiler works by spawning a background thread which periodically collects stack traces from all of the application's running threads.  Methods that take a lot of time show up frequently in these stack traces, while fast methods only appear occasionally.

Over the course of a few minutes the profiler is able to combine these stack traces into a good statistical representation of where the application is spending its time.  The Ruby agent Thread Profiler will send this data back to New Relic so that you can see what percentage of time is spent in each method call.

You can start the thread profiler from the New Relic web UI like this.

(Go to Events > Thread Profiler, click the start button)

##### HANDOFF TO BEN

Let's take a look at a Thread profile we captured from our Stealth Stars application previously.

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448/profiles/5411

Looking at this profile, we can see that the heaviest call path through our application is going through the decrypt method on our TopSecretDoc model. That's a bit surprising, given that we don't actually need the decrypted document contents to display the index page. If we expand this group of collapsed methods just above the decrypt call in the call tree, we can see that the decrypt calls are coming from ActiveSupport's run_callbacks method, and specifically, it looks like we're hitting our after_find hook each time we load a TopSecretDoc instance from the database.

Let's take a look at our model and see if we can improve this, since there's no reason to be decrypting the document bodies just to render the index page.

(Open app/models/top_secret_doc.rb)

This problem is almost the inverse of the previous one: there we were being too lazy, and here we're being too eager. We'd like to keep the convenience of not forcing code that uses this model to explicitly decrypt the document body, but not do the decryption unnecessarily. We can accomplish this by writing an accessor method for the body pseudo-attribute that lazily decrypts the first time it's called. This way, we'll defer the work of decryption until we actually need to do it.

(Make the edits to the TopSecretDoc model)

So we made this change to our TopSecretDoc model, and here's what it looked like when we deployed it:

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448/transactions?tw[end]=1366859839&tw[start]=1366858039#id=245151300

You can see that our response time for this transaction has dropped dramatically, and we're ready to add lots more documents.

### Kata 3 - (The Long Queue)

The last performance problem we're going to talk about is one that only saw in the production deployment of the Stealth Stars app.  Our secret agents are frequently in situations where they don't have reliable access to Wifi or cell networks, and as a result it's very useful for them to be able to store the sensitive information they're collecting on their cell phones and sync it to our servers when they get back on the network.

Ben and I got together and coded up a nice little espionage android app which our agents can use even when they don't have reliable network access.  We got all of our secret agents to install this on their mobile phones, and soon after our Stealth Stars web app started struggling.

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448?tw[end]=1366908619&tw[start]=1366906535

You can see here that we were having periodic spikes in response time.  We realized that the "sync data" function of the mobile app we'd given out was generating a lot of requests to our app when it came back online.  You can see here on the throughput graph that throughput will spike when an agent's app starts syncing, and this increase in throughput results in a big spike in the time each request spends in "Request Queueing".

Request Queueing on this graph is the time between when a request is received by your front end web server, and when it reaches one of your Ruby application processes.  You can usually enable this feature of New Relic with a one line change to your nginx or apache server configuration.  You just configure these servers to set a timestamp in an HTTP header before they proxy the request back to one of your application workers.  The the Ruby agent can calculate the difference between the upstream timestamp and the current time and report it as Request Queueing.

For stealth stars we're running a pool of unicorn processes behind nginx. Like most Ruby web servers unicorn is single threaded, which means that each process can only serve one request at a time.  When all of the unicorn processes are busy serving requests, nginx will queue up new requests until one of the workers becomes available.

##### HANDOFF TO BEN

Once we started seeing this pattern we were pretty sure that we had hit the capacity limitations on our current setup.  To confirm our suspicions we took a quick look at New Relic's capacity analysis report.

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448/optimize/capacity_analysis?tw[end]=1366908619&tw[start]=1366906535

You can see on the app instance analysis graph that we have three unicorn instances currently, and during our traffic spikes, we're getting all three of them working at once. The App instance busy graph shows what percentage of time workers are spending serving requests versus idling.  During our traffic spikes, we're hitting 100% utiltization, which means our three unicorn instances are spending all of their available on requests, and other requests are piling up behind them.

There are many ways to attack a problem like this. You should consider whether you can reduce the time it takes to process each request, but sometimes that's difficult. In our case, we had already optimized things pretty heavily, so we decided to just move the application to a beefier server and increase the number unicorn instances we were running, which increased the number of requests we could serve concurrently.

This is a simple change that you can make in your unicorn config file, and it looks like this:

(Open config/unicorn.conf.rb)

We bumped the number of workers up from 3 to 8. After making this change, we saw a dramatic difference in terms of response time for our application, and here's what it looked like in New Relic:

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448?tw[end]=1366909996&tw[start]=1366908196

Note that we're still seeing the same spikes in throughput, but our response time stays flat throughout them.

You can also see the difference on the capacity analysis page:

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448/optimize/capacity_analysis?tw[end]=1366909996&tw[start]=1366908196

... we're no longer bumping up against our worker capacity ceiling during traffic spikes, we're only getting to about x% utilization.

### Wrap-up

Tackling these performance problems at Stealth Stars has definitely helped us keep the organization on track.  We don't have to deal with a lot of frustrated spies complaining about how slow the app is, we've been able to keep up with as our user base and data grows. 

One thing we've noticed is that there is a lot of similarity between performance problems we've hit in this app and other apps.  Over time we've both gotten better at recognizing common performance patterns as they pop up in slightly different forms.  Practicing solving performance problems is a great way to hone your skill at recognizing them, so you can solve them quickly when you really need to.

If you want to try practicing your Rails performance skills you can check out another New Relic Ruby Code Kata which a couple other New Relic engineers developed.  It's on heroku at http://newrelic-ruby-kata.herokuapp.com/, with instructions on how to fork the code on github, deploy your own copy to heroku and enable New Relic, all for free.


If you want to take a closer look at the app we used for today's demos, or if you're looking to start your own international spy network, you can find the code at http://github.com/newrelic/stealth_stars

And lastly, if you *really* like solving these kinds of problems, and want to help us build awesome new tools to broaden and deepen the insights we can provide into Rails and Ruby performance, we're hiring!

### Questions?
