This example shows a way to connect a Ruby on Rails web application with julia through ZMQ. 

I had no previous experience with ZMQ, ruby, Rails, html or javascript before this and only basic Julia knowledge. I'm just learning as I go along and I hope this write-up migth be helpful to you. 

All feedback welcome!

Basically we create a ZMQ server in Julia that will perform some predefined calculation as instructed from a web page. In this case, supply a number and *magically* multiply it by 3 :)

This example was inspired by [a Julia-users forum topic](https://groups.google.com/forum/#!searchin/julia-users/zmq|sort:date/julia-users/umHiBwVLQ4g/8O-bC7UB2B0J), [IJulia](https://github.com/JuliaLang/IJulia.jl) and an excellent [samurails blog post on delayed_job](http://samurails.com/gems/delayed-job/). Parts of the code below is taken from that blog.

# Installation

I write this on Ubuntu 14.04. If you are using another platform, you will have to google some installation procedures yourself.

To install Ruby on Rails, I simply use [rvm](http://rvm.io):

	curl -sSL https://get.rvm.io | bash -s stable --rails

I prefer working with a PostgreSQL database, but I assume this example will work fine with MySQL or another database. Install PostgreSQL on Ubuntu (I know libpq-dev is necessary, the others I don't know):

	sudo apt-get install postgresql-client pgadmin3 postgresql postgresql-contrib libpq-dev

For running the webapp locally you need `nodejs` installed:
	
	sudo apt-get install nodejs

Of course you need Julia installed. I used v0.3 for this example. See details on [julialang.org](http://julialang.org/). In Julia you need to add the ZMQ package by simple `Pkg.add("ZMQ")` and similarly `Pkg.add("JSON")`.

Once you download this example you should install the necessary gems listed in the gem file:

	git clone https://github.com/Ken-B/RoR_julia_eg
	cd RoR_julia_eg
	bundle install

To just run the example, I think you need to initiate and migrate the database:

	rake db:create
	rake db:migrate




## follow along

I'll try to commit the outcome of each rails command and describe it in the commit message. Here's what I did.

First create a new rails app called `triple`, that will magically triple each input number you give it:

	rails new triple --database=postgresql

Include the necessary gems in the gemfile

	# Gemfile
	gem 'simple_form' #for the number input on the web page
	gem 'ffi-rzmq'	#for ZMQ client on Rails side
	gem 'delayed_job_active_record' #longer calculations need to be run in background
	gem 'daemons' #for bin/delayed_job start

then install the dependencies:
	
	cd triple
	bundle install

Initiate your database with

	rake db:create

You need to configure `simple_form` with:

	rails generate simple_form:install

For delayed_job you need to generate the table to store the job queue and actually execute the migration in the database:

	rails generate delayed_job:active_record
	rake db:migrate

Now if you start-up the server with `rails s` there's a welcome screen at (http://localhost:3000/).

---
Now let's create the app. 
We start by adding a controller.

	rails g controller Triples index new

and we set the root for our app:

	# triple/config/routes.rb

For the database we will use a `number` model with a `value` and `result` field for the input and output (= 3*input) fields, as well as a field `calculated` to indicate if the calculation has finished

	rails g model number value:float result:float calculated:boolean

In the migration file that was created we add a `false` default value to the calcaluted field. That line in `triple/db/migrate/20150209001428_create_numbers.rb` now becomes:

	t.boolean :calculated, default: false

Execute the migration in the database with the usual:

	rake db:migrate

---

###ZMQ

A quick note on [ZMQ](http://en.wikipedia.org/wiki/%C3%98MQ), a high-performance asynchronous and distributed messaging library. That's a mouthful! Basically, it allows to send messages between different processes, programs or computers, that's how I look at it. And it does most of the dirty stuff for you. You just open a socket on one end and connect to it somewhere else, and it just works.

Here I'll use JSON to structure the message. We'll use the REQ-REP pattern. We'll start a julia server as a REP service and later connect to it from Rails as a REQ.

Here's the Julia server (file `zmq_server.jl`):

	```julia
	using JSON
	using ZMQ

	const ctx = Context()
	sock = Socket(ctx, REP)
	ZMQ.bind(sock, "tcp://*:7788")

	function triple(x)
		sleep(10)
		3x
	end

	while true
		println("Server running.")
		msg = JSON.parse(bytestring(ZMQ.recv(sock)))
		@show result = triple(float(msg["value"]))
		ZMQ.send(sock, JSON.json({"result"=>result}))
	end
	```

and in the number model of the web app we will define a calculation method that connects to this server
	
	```rb
	#triple/app/models/number.rb
	require 'ffi-rzmq'
	require 'json'

	class Number < ActiveRecord::Base
	  def calculate
	    
	    context = ZMQ::Context.new
	    sock = context.socket(ZMQ::REQ)
	    sock.connect("tcp://localhost:7788")
	    
	    mgs_send = {:value => value}.to_json
	    sock.send_string mgs_send

	    msg_recv = ''
	    sock.recv_string(msg_recv)

	    result = JSON.parse(msg_recv)["result"]
	    update_column :result, result
	    update_column :calculated, true
	    
	    sock.close
	    context.terminate
	  end
	  handle_asynchronously :calculate
	  
	end
	```

Notice the `handle_asynchronously` command from `delayed_job`, which will run this in the background, non-blockins so there will not be a web page time-out.

### Webapp

Now back to the web app. This is the tricky part (at least for me).