---
title: Writing a microservice in Ruby
date: 2016-03-07 22:07 CET
category: ruby
tags: ruby, sneakers, rabbitmq, SOA
---

Everybody is talking about microservices, but I haven't seen a lot of good, comprehensive descriptions about how to actually *write* a microservice in Ruby. This may be because a significant number of Ruby developers are still most comfortable with Rails (which is not a bad thing in and of itself, but it's not all Ruby is capable of).

So I want to offer my account.

Here's the story: We want to write a microservice whose responsibility it is to send mail. It gets a message such as this:

```js
{
  'provider': 'mandrill',
  'template': 'invoice',
  'from': 'support@company.com',
  'to': 'user@example.com',
  'replacements': {
    'salutation': 'Jack',
    'year': '2016'
  }
}
```
And it knows that it has to send the invoice email template to ```user@example.com``` by substituting some variables. (We're using [mandrill](https://mandrillapp.com) as an email API provider, which sadly is going out of service soon.)

This is a perfect fit for a microservice because it's a small and focused piece of functionality with a clearly defined interface. So, that's exactly what we did at work, when we decided that our mailing infrastructure needed a clean rewrite.

Now if we have a microservice, we need a way to send it some messages. That's where a message queue comes in. There are [tons of options](http://queues.io/) here, and you may pick whatever favourite you have. We chose to go with [RabbitMQ](https://www.rabbitmq.com/), because:

- It's popular and encodes a standard (AMQP).
- It has bindings for most languages, so it's perfect for polyglot environments. I like writing Ruby (and know it better than other languages), but I don't think it's the best fit for every problem nor that it's going to be so for all future times. So if we ever have a need to write an application in, say, Elixir that sends mail through our service, it will be possible (and not too hard).
- It is very flexible and accomodates a number of workflows&mdash;simple background processing through a named queue (which I'll focus on here) or complex message exchange workflows (even RPC is possible). The website has a lot of examples.
- It comes with a useful admin panel accessible through your web browser.
- There are [hosted solutions](https://www.cloudamqp.com/) available (no affiliation). (And for development, you should be able to find it in your favourite package manager's sources.)
- It's written in Erlang and apparently those Erlang programmers know how to deal with concurrency. ;)

Putting a message into a queue with RabbitMQ is fairly easy and looks something like this:

```ruby
require 'bunny'
require 'json'

connection = Bunny.new
connection.start
channel = connection.create_channel
queue = channel.queue 'mails', durable: true

json = { ... }.to_json
queue.publish json
connection.close
```
```bunny``` is the official RabbitMQ gem and when we don't pass any options to ```Bunny.new```, it will just assume that rabbitmq runs on ```localhost:5672``` with the standard credentials. We then (after some setup) connect to a queue identified by the name "mails". If the queue doesn't exist yet, it will be created, otherwise it will connect to it. Then, we can publish any message we want directly to that queue (for example, the payload we saw above). We're using JSON here, but in fact you could be using anything you like (BSON, Protocol Buffers, whatever), RabbitMQ doesn't care.

Now, we've got the producer side covered, but we still need an application that actually takes and processes the message. For that, we used [sneakers](https://github.com/jondot/sneakers), which is a wrapper gem around RabbitMQ. It basically exposes the subset of RabbitMQ to you that you're most likely to use if you just want to do some background processing&mdash;but it's still all RabbitMQ underneath. With sneakers (which is heavily inspired by [sidekiq](https://github.com/mperham/sidekiq)), we can define a "worker" to process our mail sending requests:

```ruby
require 'sneakers'
require 'json'
require 'mandrill_api/provider'

class Mailer
  include Sneakers::Worker
  from_queue 'mails'

  def work(message)
    puts "RECEIVED: #{message}"
    option = JSON.parse(message)
    MandrillApi::Provider.new.deliver(options)
    ack!
  end
end
```
We have to specify the queue that we're reading from ("mails") and then the ```work``` method will actually consume the message. We parse the message (which we know is JSON, because that's what we agreed upon before&mdash;but again, that is your decision, RabbitMQ or sneakers don't care) and pass the message hash to some internal class that does the actual work. Finally, we must acknowledge the message or RabbitMQ will put the message back into the queue. If you need to reject a message etc., the sneakers wiki can fill you in on how to do that. We also included some logging in order to know what's going on (we'll talk later about why we log to standard output).

But one class doth not an application make. So we have to build a project structure around this&mdash;and this is something that we're not usually taught as Rails developers, because we're used to just running ```rails new``` and everything is set up for us. So I want to expand on this a little bit. This is roughly what our project tree ended up looking like:

```
.
├── Gemfile
├── Gemfile.lock
├── Procfile
├── README.md
├── bin
│   └── mailer
├── config
│   ├── deploy/...
│   ├── deploy.rb
│   ├── settings.yml
│   └── setup.rb
├── examples
│   └── mail.rb
├── lib
│   ├── mailer.rb
│   └── mandrill_api/...
└── spec
    ├── acceptance/...
    ├── acceptance_helper.rb
    ├── lib/...
    └── spec_helper.rb
```

Some of this is self-explanatory, such as the ```Gemfile(\.lock)?``` and the readme. There's also not much to say about specs, except that we used the convention to have one helper file (```spec_helper.rb```) for fast unit tests and another one (```acceptance_helper.rb```) for acceptance tests that require some more setup (e.g. mocking actual HTTP requests). The `lib` folder is also rather uninteresting for our purposes, since we already saw the ```lib/mailer.rb``` (it's the worker class we defined above) and the rest is specific to the individual service. And the ```examples/mail.rb``` is basically just the example mail enqueuing script that we saw above, kept around so we can trigger a manual test whenever needed. Now I want to specifically talk about the ```config/setup.rb``` file. This is the file that we'll always load initially (even in the ```spec_helper.rb```), so it's desirable that it doesn't do too much (or your tests will be slow). Here's what it looks like in our case:

```ruby
require 'bundler/setup'

lib_path = File.expand_path '../../lib', __FILE__
$LOAD_PATH.unshift lib_path

ENVIRONMENT = ENV['ENVIRONMENT'] || 'development'

require 'yaml'
settings_file = File.expand_path '../settings.yml', __FILE__
SETTINGS = YAML.load_file(settings_file)[ENVIRONMENT]

if %w(development test).include? ENVIRONMENT
  require 'byebug'
end
```

The most important thing we're doing here is setting the load path. First we require ```bundler/setup```, so we can henceforth require gems by name. Next, we add the ```lib``` folder of our service to the load path. This means that we can e.g. require ```mandrill_api/provider``` and it will look it up from ```<project_root>/ lib/mandrill_api/provider```. Nobody likes to deal with relative paths, so that's something you want to do. Notice that we don't get autloading like in Rails. We're also not calling ```Bundler.require``` which would actually *require* all the gems in your Gemfile&mdash;something that Rails would do for you. This means that you always *have* to require your dependencies (gems or lib files) explicitly (which I actually think is good design anyway).

Next, I like the Rails convention of having multiple environments, so we're setting it here by loading it from a UNIX environment variable called ```ENVIRONMENT``` (and defaulting to development). We're also going to need some settings (like the RabbitMQ connection options or some API keys for the services we use) and it makes sense to have them depend on the environment, so we load a YAML file for that and put that into a global variable.

Finally, we make it so that dropping ```byebug``` (the debugger for Ruby 2.x) into our code will always work in development and test by requiring it beforehand. If you're worried about the speed of that (it does take a moment) you might leave that out and only include when explicitly needed or (heresy!) you could include a monkeypatch like:

```ruby
if %w(development test).include? ENVIRONMENT
  class Object
    def byebug
      require 'byebug'
      super
    end
  end
end
```

Now, we've got a worker class and a general project structure. We just need to tell sneakers to run our worker and that's what we do in ```bin/mailer```:

```ruby
#!/usr/bin/env ruby
require_relative '../config/setup'
require 'sneakers/runner'
require 'logger'
require 'mailer'
require 'httplog'

Sneakers.configure(
  amqp: SETTINGS['amqp_url'],
  daemonize: false,
  log: STDOUT
)
Sneakers.logger.level = Logger::INFO
Httplog.options[:log_headers] = true

Sneakers::Runner.new([Mailer]).run
```
Notice that this is an executable (the shebang at the top), so we can just run it without the ```ruby``` command. We first load our setup file (for this we have to use a relative path) and then all the other stuff that we need, including our mailer worker class.

The important part here is where we configure sneakers: The ```amqp``` argument accepts a URL that specifies the RabbitMQ connection which we load from the settings. We also tell sneakers to run in the foreground and log to standard ouput (more on that in a bit). Then we basically give sneakers an array of worker classes and tell it to run them. We also took the liberty to include some logging, so we could see what's going on. The [httplog](https://github.com/trusche/httplog) gem will just log all outgoing requests which is quite useful if you're talking to an external API (here we're making it also log the HTTP headers, which it doesn't do by default).

Now if you run ```bin/mailer``` you will see something like the following:

```
... WARN: Loading runner configuration...
... INFO: New configuration:
#<Sneakers::Configuration:0x007f96229f5f28 ...>
... INFO: Heartbeat interval used (in seconds): 2
```
(though the actual output is much more verbose!)

If you keep it running and then run our enqueue script from above in another terminal window, you'll get something like:

```
... RECEIVED: {"provider":"mandrill","template":"invoice", ...}
D, ... [httplog] Sending: POST
https://mandrillapp.com:443/api/1.0/messages/send-template.json
D, ... [httplog] Data: {"template_name":"invoice", ...}
D, ... [httplog] Connecting: mandrillapp.com:443
D, ... [httplog] Status: 200
D, ... [httplog] Response:
[{"email":"user@example.com","status":"sent", ...}]
D, ... [httplog] Benchmark: 1.698229061003076 seconds
```
(again shortened here) which is pretty informative, especially in the beginning, but of course you can later reduce log noise as needed.

This gives us the basic project structure, so what's left to do? Well, here comes the hard part: Deployment.

There's lots of things you have to take care of when you deploy a microservice (or, in general, any application), including the following:

- You want to daemonize it (i.e., it should run in the background). We could have done this when setting up sneakers above, but I prefer not to&mdash;in development, I like to be able to see the log output and to be able to kill the process with ```CTRL-C```.
- You want to have proper logging. This include somehow making sure that your logfiles don't end up filling your whole disk or becoming so unmanageably huge that grepping through them takes forever (e.g. log rotation).
- You want it to restart both when you have to restart your server for whatever reason and when it crashes because who knows what.
- You want to have standard commands for starting / stopping / restarting when needed.

You could do this all by yourself in Ruby, but I think there's a better solution: Use something that's built to handle those tasks, i.e. your operating system (Mike Perham, the creator of sidekiq, [agrees with me](http://www.mikeperham.com/2014/09/22/dont-daemonize-your-daemons/)). In our case, this meant ```systemd``` since that's what runs on our servers (and on most Linux systems nowadays), but I really don't want to pick a fight here. Upstart or daemontools probably work just as fine.

For systemd to run our microservice we need to generate some config files. This could be done by hand, but I prefer to use a tool called [foreman](http://blog.daviddollar.org/2011/05/06/introducing-foreman.html). With foreman, we can specify all the processes that our app needs to spin up in a ```Procfile```:

```
mailer: bin/mailer
```

Here, we only have one process, but you could have several ones. We have a process that we identify as "mailer" and it should run the ```bin/mailer``` executable. Now, the nice part about foreman is that it is able to export this configuration to a number of init systems, including systemd. For example, out of this simple Procfile it will actually create a bunch of files; this is because, as I mentioned earlier, we could have multiple processes specified in the Procfile, so the different files specify a hierarchy of dependencies. We will have a top-level ```mailer.target``` file that depends on a ```mailer-mailer.target``` file (if we had multiple processes in the Procfile, ```mailer.target``` would depend on multiple sub-targets). This is in turn dependent on ```mailer-mailer-1.service``` (we could have more than one such file by explicitly setting the concurrency level to a value higher than one). That last file looks something like this:

```
[Unit]
PartOf=-.target

[Service]
User=mailer_user
WorkingDirectory=/var/www/mailer_production/releases/16
Environment=PORT=5000
Environment=PATH=
/home/deploy/.rvm/gems/ruby-2.2.3/gems/bundler-1.11.2:...
Environment=ENVIRONMENT=production
ExecStart=/bin/bash -lc 'bin/mailer'
Restart=always
StandardInput=null
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=%n
KillMode=process
```
The exact details are not so important, but here we can see that we specify the user the service should run as, the working directory, the command to run to start the service, that we should always restart after a failure and that we log to the system logger. We also set some environment variables, including the ```PATH```, something I'll talk about in a bit.

With this, we get all of the behaviour that we wanted earlier. The job will run in the background and restart after a failure. You can also make it run when the system boots up by running ```sudo systemctl enable mailer.target```. And the log output, that we're just sending to standard output, will get redirected to the system logger. In systemd's case, this is typically ```journald```, which is a binary logger (where the issue of rotating your logs doesn't arise anymore). Inspecting the log output for our mailer service can be done with:

```
$ sudo journalctl -xu mailer-mailer-1.service
-- Logs begin at Thu 2015-12-24 01:59:54 CET, end at ... --
Feb 23 10:00:07 ... RECEIVED: {"from": ...}
...
```
You can give ```journalctl``` more options and e.g. filter by date.

In order to have foreman generate the ```systemd``` files, we have to set up the export process in our deployment. Now, I don't know whether you use Capistrano 2 or 3 or something else (like [mina](http://nadarei.co/mina/)), so I'm just going you to show the respective shell commands that you'll have to incorporate. The hardest task somehow is always getting your environment variables set up correctly. In order for foreman to write the variables we saw above to the startup scripts, we can put them into a ```.env``` file first, by running this from the deployed project root:

```
$ echo "PATH=$(bundle show bundler):$PATH" >> .env
$ echo "ENVIRONMENT=production" >> .env
```
(I'm ignoring the ```PORT``` variable here&mdash;foreman generates it automatically, and we don't even need it for our service.)

Then we tell foreman to export to systemd while reading these variables in from the ```.env``` file we just created:

```
$ sudo -E env "PATH=$PATH" bundle exec foreman\
  export systemd /etc/systemd/system\
  -a mailer -u mailer_user -e .env
```
That's a long command, but it boils down to running ```foreman export systemd```, while specifying the directory that the files should be put in (```/etc/systemd/system``` is, as far as I can tell, a "standard" directory for it), the name of the target, the user it should run as and the environment file it should load.

Then we reload everything:

```
$ sudo systemctl daemon-reload
$ sudo systemctl reload-or-restart mailer.target
```

And then, we enable the service in order to make it always run when the server boots up:

```
$ sudo systemctl enable mailer.target
```

After that, our service should be up and running on our server, ready to accept any incoming messages.

I have covered a lot of ground in this post, but it is my hope that I showed some of you the big picture behind writing and deploying a microservice. Obviously, you'll have some research to do if you really want to follow along and do this yourself, but I think I have given you some pointers to technologies you could be looking at.

We wrote our mailer service basically like this a couple months ago and so far we're pretty happy with the results. The fact that the mailer is isolated with a clearly defined API and is well-tested independently gives us a lot of confidence in it doing what it's supposed to do. And having a sane restart behaviour is a deal-breaker for us&mdash;we still have some sidekiq workers that occasionally die and while we fixed the issue by adding *monit* on top of it, it feels much better to fully use the tools your operating system provides.
