
### Who are we

I'm Sam Goldstein & I'm Ben Weintraub.  We both work at New Relic, which is an application performance monitoring service that can gives visibility into how your Ruby applications are performing in production.  We work specifically on the Ruby agent which you may know as the newrelic_rpm gem.  This is the code that runs in your app and gathers performance metrics which are sent back to our severs.

### What is a Performance Kata?

Kata (型 or 形 literally: "form"?) is a Japanese word describing detailed choreographed patterns of movements practised either solo or in pairs.  Kata originally were teaching and training methods by which successful combat techniques were preserved and passed on.

Dave Thomas coined the term Code Kata to describe a coding problem that you can solve over and over again to practice your programing and hone your problem solving reflexes.

At New Relic we've been applying this concept to performance.  Performance is an important feature of any software, and when you solve performace problems you realize that you end up solving similar problems over and over again, in different projects and contexts.  Practicing your ability to recognize the shape of these common problems helps you become a better programmer and deliver better software.

If you want to try this on your own you should check out http://newrelic-ruby-kata.herokuapp.com/ which is an app you can easily deploy for free to heroku which contains a set of performance puzzles and instructions to guide you through them.

Today though, we're going to go through another set of performance problems that we've hit on a Rails application we have been developing. We'll show you how we identified these problems using New Relic and how we fixed them.

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
https://rpm.newrelic.com/accounts/319532/applications/2107448

This is the main overview page for the whole Stealth Stars application.  This is useful for when you want to keep an eye over how you're whole app is performing, but for this problem we wanted to dive into a specific transaction - the Missions index page.  To do this we jumped in to "Web Transactions" tab a selected the transaction we're interested in from the list.

Go to:
https://rpm.newrelic.com/accounts/319532/applications/2107448/transactions?tw[end]=1366847813&tw[start]=1366846725#id=245140449

Looking at this page you can see that the response time for this action is around XXX ms, which is pretty slow, considering that we're only showing about 100 missions.  From the breakdown graph we can see that we're spending a lot of time rendering the index template and that we're spending a lot of time in XXX#find.

To really dive into what was causing this slowness we dove into a specific Transaction Trace.  When New Relic notices a slow request it will capture a Transaction Trace which shows a detailed sequence of the events used to render that request.

Here's a transaction trace for one request to the Missions overview page.
Go to:
https://rpm.newrelic.com/accounts/319532/applications/2107448/transactions?tw[end]=1366847813&tw[start]=1366846725#id=245140449&app_trace_id=964667539&tab-detail_964667539=trace_details


##### HANDOFF TO BEN

Looking at this transaction trace for the MissionsController#index, you can see a chronological view of the events that happened during this particular request, and how long each of them took. Starting from the top, we immediately see a big chunk of time that takes (whatever) ms for rendering our index.html.erb template.

That's pretty strange, so let's drill down further. This red row nested under the template rendering indicates that we've got 1000 calls to Operative#find_by_sql originating from our template - a classic N+1 queries problem. Let's take a look at the code and see if we can fix it.

(Open app/views/missions/index.html.erb)

Here's our template for the missions index page - you can see we're looping over the @missions variable assigned by the controller action, and then hitting the operatives relation on each mission to load all of the operatives and iterate over them. Each one of these calls to Mission#operatives is resulting in a separate database query. ActiveRecord makes it easy to fix this - let's jump over to the controller to see how.

(Open app/controllers/missions_controller.rb)

What we'd like to do is tell ActiveRecord to eagerly load the operatives association here, since we know we're going to need it to render the template. There are multiple ways to accomplish this, but he easiest one to demonstrate is just to chain in an includes(:operatives) call here into our query.

One of the most gratifying experiences to have with New Relic is to deploy your change to production, and see the graphs shift dramatically after your change. Here's what it looked like when we made this change to our Stealth Stars application in production:

Go to:
https://rpm.newrelic.com/accounts/319532/applications/2107448/transactions?tw[end]=1366848781&tw[start]=1366846981#id=245140449

And just for good measure, here's a transaction trace showing what this transaction looks like after our change:

Go to:
https://rpm.newrelic.com/accounts/319532/applications/2107448/transactions?tw[end]=1366848670&tw[start]=1366848023#id=245140449&app_trace_id=964698861&tab-detail_964698861=trace_details

You can see that our 1000 calls to Operative#find_by_sql are gone, and our overall response time is much improved. We're ready to scale up to thousands more missions and operatives!

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


Looking at this thread profile, we can see that the heaviest call path through our application is going through the decrypt method on our TopSecretDoc model that we were looking at previously. But why are we decrypting the document bodies at all in order to display the index page? This apge doesn't actually show the document bodies, just the titles, which are stored unencrypted. Let's take another look at our model.

(Open app/models/top_secret_doc.rb)

You can see that we've got an after_find hook here that's eagerly decrypting our document bodies. This means the body gets decrypted (which is a relatively expensive operation) every time we load one of these models via ActiveRecord, even if we never directly reference the body attribute.

Fixing this is actually pretty easy: we can make the body synthesized attribute lazily decrypted by adding a body method that assigns to it, and removing the after_find hook, like this:

(Make the edits to the TopSecretDoc model)

Here's what it looked like in New Relic when we made this change to our production application:

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448/transactions?tw[end]=1366859839&tw[start]=1366858039#id=245151300

### Kata 3 - (The Long Queue)

The last performance problem we're going to talk about is one that only saw in the production deployment of the Stealth Stars app.  Our secret agents are frequently in situations where they don't have reliable access to Wifi or cell networks, and as a result it's very useful for them to be able to store the sensitive information they're collecting on their cell phones and sync it to our servers when they get back on the network.

Ben and I got together and coded up a nice little espionage android app which our agents can use even when they don't have reliable network access.  We got all of our secret agents to install this on their mobile phones, and soon after our Stealth Stars web app started struggling.

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448?tw[end]=1366908619&tw[start]=1366906535

You can see here that we were having periodic spikes in response time.  We realized that the "sync data" function of the mobile app we'd given out was generating a lot of requests to our app when it came back online.  You can see here on the throughput graph that throughput will spike when an agent's app starts syncing, and this increase in throughput results in a big spike in the time each request spends in "Request Queueing".

Request Queueing on this graph is the time between when a request is received by your front end web server, and when it reaches one of your Ruby application processes.  You can usually enable this feature of New Relic with a one line change to your nginx or apache server configuration.  You just configure these servers to set a timestamp in an HTTP header before they proxy the request back to one of your application workers.  The the Ruby agent can calculate the difference between the upstream timestamp and the current time and report it as Request Queueing.

For stealth stars we're running a pool of unicorn processes behind nginx. Like most Ruby web servers unicorn is single threaded, which means that each process can only serve one request at a time.  When all of the unicorn processes are busy serving requests, nginx will queue up new requests until one of the workers becomes available.

##### HANDOFF TO BEN

Here's what that looks like in the nginx configuration file:

(Open config/nginx.conf)

This directive tells nginx to write the X-Request-Start header into incoming requests, with a timestamp in microseconds since the beginning of the UNIX epoch. The Ruby agent recognizes this header and records the time interval between when nginx saw the request and when it reached our controller in Rails as the queue time.

We used the capacity graphs on the capacity analysis tab to help further quantify the issues we were seeing with queue time on our app. Let's see how that looked.

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448/optimize/capacity_analysis?tw[end]=1366908619&tw[start]=1366906535

You can see on the app instance analysis graph that we have three unicorn instances currently, and during our traffic spikes, we're getting all three of them working at once. Looking at the App instance busy graph, we can get an idea of the percentage of time our instances are actually spending servicing requests (versus idling). One thing that's important to note about our setup is that each unicorn worker is only able to service a single request at a time. During our traffic spikes, we're hitting 100% utiltization, which means our three unicorn instances are spending all of their available time servicing requests, with almost no idle time in between requests. We have no headroom during those traffic spikes, and thus, requests start piling up in the queue.

There are many ways to attack a problem like this, and generally you should first consider whether you can reduce the time it takes to process each request, but sometimes that's difficult. In our case, we had already optimized the API action that was being hit pretty heavily, so we decided to try just bumping up the number of unicorn instances available to service requests, thus increasing the number of simultaneous requests we could service.

This is a simple change that you can make in your unicorn config file, and it looks like this:

(Open config/unicorn.conf.rb)

We bumped the number of workers up from 3 to 8. After making this change, we saw a dramatic difference in terms of response time for our application, and here's what it looked like in New Relic:

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448?tw[end]=1366909996&tw[start]=1366908196

The difference in the capacity analysis report is also striking:

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448/optimize/capacity_analysis?tw[end]=1366909996&tw[start]=1366908196

... you can see that we're no longer bumping up against our worker capacity ceiling during traffic spikes, we're only getting to about x% utilization.

### Wrap-up

We found New Relic to be an invaluable tool in scaling our spy organization's web app, and we think it can be invaluable for your app, too. There are a lot of great tools out there for understanding Ruby performance issues, with varying degrees of invasiveness and insight, and we see New Relic as an important part of this greater tool ecosystem. There can be many causes for performance problems - weird interactions between compontents sometimes occur only in production and only at scale, and we think New Relic's key advantage as a tool is that you can use it in production.

It's always a great feeling to see the impact of a fix on your production system via the data that New Relic reports, so we've tried to show you a few simple examples of what this looks like. If you enjoyed practicing your Rails performance skills by working through these issues, we highly encourage you to check out the official New Relic Ruby Code Kata on GitHub: http://github.com/newrelic/newrelic-ruby-kata

If there's anything you wanted to dig into deeper from our demos today, or if you're looking to start your own international spy network, you can also find the code we used for the app in this talk on GitHub at http://github.com/newrelic/stealth_stars

And lastly, if you *really* like solving these kinds of problems, and want to help us build awesome new tools to broaden and deepen the insights we can provide into Rails and Ruby performance, we're hiring!

### Questions?
