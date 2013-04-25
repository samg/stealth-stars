
### Who are we

I'm Sam Goldstein & I'm Ben Weintraub.  We both work at New Relic, which is an application performance monitoring service that can give you a lot of visibility into how you Ruby applications are performing in production.  We work specifically on the Ruby agent which you may know as the newrelic_rpm gem.  This is the code that runs in your app and gathers performance metrics which are sent back to our severs.

### What is a Performance Kata?

Kata (型 or 形 literally: "form"?) is a Japanese word describing detailed choreographed patterns of movements practised either solo or in pairs.  Kata originally were teaching and training methods by which successful combat techniques were preserved and passed on.

Dave Thomas coined the term Code Kata to describe a coding problem that you can solve over and over again to practice your programing and hone your problem solving reflexes.

At New Relic we've been applying this concept to performance.  Performance is an important feature of any software, and when you solve performace problems you realize that you end up solving similar problems over and over again, in different projects and contexts.  Practicing your ability to recognize the shape of these common problems helps you become a better programmer and deliver better software.

If you want to try this on your own you should check out http://newrelic-ruby-kata.herokuapp.com/ which is an app you can easily deploy for free to heroku which contains a set of performance puzzles and instructions to guide you through them.

Today though, we're going to go through another set of performance problems that we've hit on a Rails application we have been developing. We'll show you how we identified these problems using New Relic and how we fixed them.

### Stealth Stars Inc.

Like we said at the beginning of this talk, Ben and I work on the Ruby agent team at New Relic.  But in our spare time we've been bootstrapping a secret intellegence contracting organization called "Stealth Stars Inc."  We thought it would be really cool to run a global network of secret agents.  As we've been growing this business we've been writing software to help us run it, but we've hit a few performance problems along the way.  We're going to walk through three of these performance problems and show you how we identified and fixed them in our codebase.

### Kata 1 - The big loop

At Stealth Stars we've been hiring a lot of new operatives and taking on a lot of new missions.

As our network of operatives grew we realized we were having trouble keeping track of which operatives were assigned to which missions. Our operatives are so talented and on top of it that they are often assigned to multiple missions at once. So, we built a missions overview page in our application to list all of our active missions, along with their priority, and the list of operatives assigned to each one. Here's what it looks like:

(Open http://localhost:8080/missions)

As you can see, we've got our mission codenames on the left, and the assigned operatives for each mission on the right. You can probably imagine the code for implementing this in your head - we're using Mission and Operative models with a has_and_belongs_to_many relationship between them. Pretty straightforward, and Rails makes this dead simple.

As the number of missions and operatives continued to grow, however, we noticed this page was getting a little pokey. We hooked our application up to New Relic's performance monitoring tool, and this allowed us to quantify the performance problem a bit.

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448

Here's the overview page for our whole Stealth Stars application. This is great for looking at the performance of our application as a whole, but in this case we want to focus on one particular transaction - the Missions index page. We can get there by going to the 'Web Transactions' tab and selecting the transaction we're interested from the list on the left.

Go to:
https://rpm.newrelic.com/accounts/319532/applications/2107448/transactions?tw[end]=1366847813&tw[start]=1366846725#id=245140449

Looking at this, you can see the response time for this action is around (whatever) ms, which is not great. Looking at the overview graph, we can also see that the majority of the time seems to be in Operative#find, and in template rendering. Those are useful hints, but they still don't tell us exactly what's going on with our transaction.

New Relic can also capture Transaction Traces, which are detailed breakdowns of the exact sequence of events for a given transaction. Transaction Traces are by default only captured for transactions that take longer than a certain threshold, and only for a sampling of transactions in order to keep the overhead of gathering them low.

Go to:
https://rpm.newrelic.com/accounts/319532/applications/2107448/transactions?tw[end]=1366847813&tw[start]=1366846725#id=245140449&app_trace_id=964667539&tab-detail_964667539=trace_details

## HANDOFF

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

Around the time that we fixed the issue with our missions index page, we realized that we also had a need to enable our operatives to securely store top secret documents that they were gathering or authoring in the field. We're not quite done building this system yet, but one thing that was really important to us was that the unencrypted document contents never touched the disk on our databaase server. This was important to us because we push our database backups to S3 for archival, and we wanted to ensure they were safe out there.

Of course, with all the thought that went into our document encryption mechanism, we haven't actually built the authentication layer on top of it, but that's coming in the next sprint or two.


Here's our current work-in-progress system:
http://localhost:8080/top_secret_docs

Anyhow, in order to make it easier to work with these encrypted documents, we made use of the ActiveRecord object lifecycle hooks to transparently handle encryption and decryption of document bodies.

We have a list all the docs here, and when you click on a particular one, you can see both the encrypted and decrypted versions of the document body. It looks like our operatives have been gathering a lot of intelligence from the old UNIX 'fortune' program.

You probably noticed that the index page for our top secret docs loaded pretty slowly. Since our operatives are paid hourly, it's important to us that they not spend their time waiting for pages in our application to load.

Just for some context, let's take a quick look at how we implemented our document encryption system in concert with ActiveRecord.

(Open app/models/top_secret_doc.rb)

We wanted the encryption system to be as transparent as possible, because we really care about the maintainability of our application, so we used ActiveRecord's object lifecycle hooks to automatically encrypt document bodies before saving them to the database, and decrypt them upon retrieval from the database. The encrypted_body attribute is the only one that's actually saved in the DB, and the 'body' attribute is synthesized by our hooks.

Since we were already using Rails, we decided to use the handy ActiveSupport::MessageEncryptor class to handle encryption and decryption. This has the added benefit of allowing us to append a cryptographic signature to the encrypted the document bodies as well, giving us a gaurantee that they have not been modified. The whole system works pretty well: you don't need to think about the encryption when creating new documents or when loading them from the database, you just use the body attribute as if it were real. You'll also notice that we're doing multiple rounds of encryption because we wanted to be *extra* secure.

So let's take a look at a transaction trace for this page and see if we can spot where the performance issue is.

(Open TT page for TopSecretDocsController#index)

This isn't as helpful as the last trace we looked at - notice this big chunk here is labelled as 'Application code'. By default, the Ruby agent only instruments certain well-known events in order to keep the overhead of tracing low. There are a few ways of getting more detail here, including telling the agent about other methods in our application code that we want it to trace, but sometimes you just have no idea where to start. In cases like that, it's sometimes quicker to use another relatively new feature: the Thread Profiler.

The thread profiler periodically gathers backtraces for all threads within your application, and aggregates those backtraces together into a profile that you can view through the New Relic website in order to get a picture of where your application is spending most of its time. It's a great tool to get a quick overview of what the hottest code paths through your application are, and with that information in hand, you can go back and add custom instrumentation to key parts of your application.

You can start the thread profiler from the New Relic web UI like this.

(Go to Events > Thread Profiler, click the start button)

Let's take a look at a Thread profile we captured from our Stealth Stars application previously.

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448/profiles/5411

### HANDOFF

Looking at this thread profile, we can see that the heaviest call path through our application is going through the decrypt method on our TopSecretDoc model that we were looking at previously. But why are we decrypting the document bodies at all in order to display the index page? This apge doesn't actually show the document bodies, just the titles, which are stored unencrypted. Let's take another look at our model.

(Open app/models/top_secret_doc.rb)

You can see that we've got an after_find hook here that's eagerly decrypting our document bodies. This means the body gets decrypted (which is a relatively expensive operation) every time we load one of these models via ActiveRecord, even if we never directly reference the body attribute.

Fixing this is actually pretty easy: we can make the body synthesized attribute lazily decrypted by adding a body method that assigns to it, and removing the after_find hook, like this:

(Make the edits to the TopSecretDoc model)

Here's what it looked like in New Relic when we made this change to our production application:

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448/transactions?tw[end]=1366859839&tw[start]=1366858039#id=245151300

### Kata 3 - (The Long Queue)

Our last example touches on some issues related to deployment, outside of our actual application code. At some point some gadget-head within our organization decided it would be cool to give all of our agents a smartphone app that would allow them to periodically download information about all active operatives and missions to their phones, so that they'd always be up to date on things.

So, we added a simple API action to our app that returns this wall of JSON:

(http://localhost:8080/operatives.json)

That action is consumed by the smartphone app, which polls our servers periodically for updates. After giving this app to all of our field operatives, we started noticing some periodic spikes in our application's response time.

Go to
https://rpm.newrelic.com/accounts/319532/applications/2107448?tw[end]=1366908619&tw[start]=1366906535

You can see that these spikes mainly seem to be caused by 'Request Queueing' time. What does that mean? Well, our application is using what's become a pretty standard deployment setup in the Rails world: we've got nginx as a front-end web server, feeding requests to a few unicorn worker processes on the backend. When we deployed this setup, we made a quick configuration change to nginx to have it write a timestamp into the request headers of incoming requests before passing them along to unicorn backend workers.

### HANDOFF

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
