## Getting Started with Redis and Resque

Redis: <http://redis.io/>   
Resque: <https://github.com/defunkt/resque>   


### Redis

Redis is a key-value store on steroids. To install:

			brew install redis

Do the additional setup - Like Postgres, redis should just always be running.

			mkdir -p ~/Library/LaunchAgents
			cp /usr/local/Cellar/redis/2.4.2/io.redis.redis-server.plist ~/Library/LaunchAgents/
			launchctl load -w ~/Library/LaunchAgents/io.redis.redis-server.plist

Redis listens on port 6379. So you can exercise it with telnet, and type plain text commands to set the value for a key, and then later get the value associated with that key:

			[jgn@jgnmbp resque-examples]$ telnet 127.0.0.1 6379
			Trying 127.0.0.1...
			Connected to localhost.
			Escape character is '^]'.
			set user1 john
			+OK
			get user1
			$4
			john
			                                                         NOTE: press ctrl-]
			telnet> quit
			Connection closed.

(The “$4” is providing the length of the return value in number of bytes; for more on this, see http://redis.io/topics/protocol)

Were you to shut down Redis right now, and restart it, you’d still be able to get the value at key user1. That’s because Redis, unlike memcached, is persistent. Backups can be done at any time while Redis is running with a simple “cp” command.

An easier way to deal with redis is to use the CLI, redis-cli:

			[jgn@jgnmbp resque-examples]$ redis-cli
			redis 127.0.0.1:6379> get user1
			"john"
			redis 127.0.0.1:6379> quit

Where Redis really shines is in its datatypes. memcache is pretty much only a key-value store. But Redis lets you manipulate higher-order objects, such as lists (http://redis.io/commands#list) and sets (http://redis.io/commands#set). It can also handle transactions and has an API for pub/sub.


### Resque

Resque manages job queues. From one program, you can put a task onto a queue. Then another program can pop the task off the queue and perform the task.

To see this:

			gem install resque
			git clone git@github.com:tuker/resque-examples.git
			cd resque-examples

Example code to put tasks onto a queue called “compute”:

			require 'rubygems'
			require 'resque'

			require './computer_worker'

			# class ComputerWorker
			#   @queue = :computer
			# end

			class Computer
			  def add(a, b)
			    Resque.enqueue(ComputerWorker, :+, a, b)
			  end
			  def multiply(a, b)
			    Resque.enqueue(ComputerWorker, :*, a, b)
			  end
			end
			computer = Computer.new
			computer.add(5, 10)
			computer.multiply(10, 0.5)

Example code to perform a task:

			require 'rubygems'
			require 'resque'

			class ComputerWorker
			  @queue = :compute

			  def self.perform(operation, a, b)
			    puts "Work started for #{operation} on #{a} and #{b}"
			    sleep 5
			    puts "#{a} #{operation} #{b} = #{a.send(operation, b)}"
			  end
			end

First let’s run the Resque worker which will pop items off of the “compute” queue:

			QUEUE=compute rake resque:work

In a 2nd window:

			ruby computer.rb

Watch both windows. Notice that the program in the 2nd window finished right away. The tasks were queued and it finished. While in the 1st worker window, you see the jobs getting completed. There’s a sleep call in the worker code to slow it down so that it’s easier to observe (in a moment, we’ll inspect Redis while Resque is running).

Now, in the 2nd window, do “ruby computer.rb” again. Immediately after it exits, type “resque list” -- this will give you some insight into whether Resque is processing anything.

			[jgn@jgnmbp resque-examples (master)]$ resque list
			jgnmbp.local:10456:compute (working)

When the worker is done, re-run the “resque list” command in the 2nd window, and you should see that the queue is not longer working, but idle:

			[jgn@jgnmbp resque-examples (master)]$ resque list
			jgnmbp.local:10456:compute (idle)

Finally, Resque comes with a nice web-based interface. Type:

			resque-web

This will pop up a Sinatra app that can report on your queues.

In the 2nd windows, type “ruby computer.rb” and immediately afterwards, refresh the Sinatra app a few times to watch.

One oddity of Resque is that the assumption is that the program that queues the worker has access to the class that does the processing. But this doesn’t have to be so.

In the computer.rb file, comment out “require './computer_worker'” and un-comment-out the commented-out three lines that define a ComputerWorker class.

Re-run computer.rb. As you can see, the jobs do get enqueued properly. This means that the the “client” code doesn’t have to have access to the processing code. It only needs a “shell” class that defines the queue name.

To get some insights into what Resque is doing, open a 3rd window and type

			redis-cli

then

			keys *

Interesting, eh? You should see a key that looks like resque:worker:jgnmbp.local:10675:compute:started - Use the get command to check the value. It’s a datetime. Now try

			get resque:workers

Ah, didn’t work. To find out the type of value, do

			type resque:workers

It’s a set. If you go to the doc for Redis sets (http://redis.io/commands#set) you’ll see that smembers is a good command to try . . .

			smembers resque:workers

Run your computer.rb again, and, right after it starts, type into your redis-cli the following:

type resque:queue:compute

And then immediately do . . .

			lindex resque:queue:compute 0

As you can see, Resque manages its queues using the primitive data structures of Redis.
