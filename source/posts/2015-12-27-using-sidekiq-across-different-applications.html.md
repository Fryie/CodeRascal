---
title: Using Sidekiq across different applications
date: 2015-12-27 16:00 CET
category: ruby
tags: ruby, sidekiq, SOA
---

In a project I was working on, we wanted to split the big monolith application we had into several smaller repositories. In particular, we wanted the user-facing part of our application to be completely separate from the administrative backend. This is all nice and well, but then we had to consider how we can still make sure that jobs such as sending email are properly enqueued and executed. Since we were using Sidekiq, which just runs atop of a Redis store, this is not hard to do in principle: You just put some items in your queue from one repository and read from that same queue in the other repository.

READMORE

Let's quickly review how enqueuing and dequeuing works in Sidekiq. If you have the following worker in your project:

```ruby
class EmailWorker
  include Sidekiq::Worker
  sidekiq_options queue: 'email', retry: false

  # dequeuing and executing job
  def perform(recipient, message)
    send_email recipient, message
  end

  private

  def send_email(recipient, message)
    # code omitted
  end
end
```

Then at some other point in your code you can enqueue a job

```ruby
EmailWorker.perform_async('test@example.org', 'Hello World')
```

If you're in Rails, you can just run ```bundle exec sidekiq``` from your project root and Sidekiq will start and automatically pick up any work for which there is a corresponding worker in the project (for more details, check the [Sidekiq Wiki](https://github.com/mperham/sidekiq/wiki)).

But what happens when you have two different repositories, one which enqueues jobs and one which dequeues them? (In practice both projects could be enqueueing and dequeueing jobs, but let's keep it simple.) If the ```EmailWorker``` is only in the project that actually executes the job, you cannot call ```EmailWorker#perform_async``` in the other repository. But remember that Sidekiq just runs atop of Redis - it just pushes jobs to some queues and reads from those queues. Instead of the more "magical" ```perform_async``` syntax, you can also drop down to a slightly lower level and do the following to enqueue a job:

```ruby
require 'sidekiq'
Sidekiq::Client.push(
  class: 'EmailWorker',
  queue: 'email',
  retry: false,
  args: ['test@example.org', 'Hello World']
)
```

Notice that the ```class``` option is just a string here. This is nice, since it keeps the code decoupled from the actual implementation of the ```EmailWorker```. The enqueuing repository does not need to be concerned with how the actual worker is implemented in another project - in fact it's even possible that the ```EmailWorker``` doesn't exist at all.

However, this approach also has a number of drawbacks:

- First, if you enqueue jobs for the email worker in multiple places, you increase the risk of introducing a typo somewhere. In the default single-repository Sidekiq setup you reference worker classes by their constants, so your code will immediately throw an error if the wrongly spelt ```EmialWorker``` could not be found. This is a kind of type safety (if one can abuse this word here) that you don't get in this case. If you run ```Sidekiq::Client.push class: 'EmialWorker' ```, the code will execute just fine, but the actual worker will never run.

- Second, Sidekiq allows you to introduce a number of default options in your worker definitions. For example, above we specified that the ```EmailWorker``` should use the "email" queue and that it should not try to rerun failed jobs. Specifying these options is actually a responsibility of the enqueuing code, not of the dequeueing code, so if we use the solution with ```Sidekiq::Client.push``` we will have to specify the "queue" and "retry" options every time we want to send an email

- And finally, the code is just much less readable this way.

Of course, the simplest way to mitigate these drawbacks would be to simply create a method ```send_email``` that encapsulates this behaviour. However, what if there was a way to just reuse the normal Sidekiq syntax without having to share the worker classes?

Enter worker proxies.

The solution is quite simple actually: Instead of using the worker classes directly, a project that wants to enqueue jobs for specific workers just defines *proxy* classes. These proxy classes follow a particular naming convention: An ```EmailWorkerProxy``` matches an ```EmailWorker```. They can also define their default Sidekiq options for queueing, retry behaviour etc. just like regular old Sidekiq workers. This is what a worker proxy could look like:

```ruby
class EmailWorkerProxy
  include WorkerProxy
  sidekiq_options queue: 'email', retry: false
end
```

Notice that the class doesn't include any ```#perform``` method: That is up to the actual worker class for which this class is only a proxy. Now somewhere in your code you could just run:

```ruby
EmailWorkerProxy.perform_async('test@example.org', 'Hello World')
```

There is just one drawback to this solution: It is not provided by Sidekiq. However, it turns out that if we muck a little with Sidekiq's internals, it's actually not that hard to implement the ```WorkerProxy``` module ourselves. This is what it looks like:

```ruby
module WorkerProxy
  def self.included(base)
    base.send(:include, Sidekiq::Worker)
    base.extend ClassMethods
  end

  module ClassMethods
    # override
    def client_push(item)
      # get sidekiq options defined in proxy
      extended_item = get_sidekiq_options.merge(item)

      # strip 'Proxy' from class name
      proxied_class_name = to_s[0..-6]

      super(extended_item.merge 'class' => proxied_class_name)
    end
  end
end
```

What this does is basically setting up the including class as a normal Worker class by including ```Sidekiq::Worker```, but then overriding the [internal #client_push method](https://github.com/mperham/sidekiq/blob/master/lib/sidekiq/worker.rb#L84). Besides merging the default options defined in the WorkerProxy file with any options passed in directly, it just replaces the 'class' option - which would be e.g. ```EmailWorkerProxy``` with the actual worker class, in this case ```EmailWorker```.

Now in all fairness, utilising a private API is not really solid design and has the risk of breaking with future releases. It does work well in our current projects, though, and it has a certain elegance, at least as a proof of concept. It would be nice if Sidekiq offered such an option by default.
