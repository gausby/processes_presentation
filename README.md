Processes
=========

Let me be honest here: I have a background in Node.js. For some reason I have a great respect of processes; they are fragile, and has to be handled with great respect. If one dies everything is lost. Better reboot that server. For some reason I have carried that fear onto Elixir, and from talking to others in the Elixir community I am not the only one. I know a couple of people who has released Elixir modules on Hex, even some that are quite popular, but none of them has a supervision tree. We skip those processes, we will not touch them with a ten-foot pole.

How come? Building modules in Elixir that transform data is super nice and all, but asking some of the old Erlang programmers about the coolest thing about the BEAM, and they will most certainly point at processes. Some might even say that you might as well program Python, or Ruby, if you do not use processes.

In this talk I will try to dissect processes in an attempt to demystify them.


The Basics
----------
We have a couple of Erlang primitives for dealing with processes: `spawn/1` is one of them. When a process is spawned it will return a Process Id, shortened Pid.

```elixir
iex(1)> my_process = spawn(fn -> :ok end)
#PID<0.62.0>
```

When a spawned process is done with its work it will close itself down. We can ask if a process is done with the `Process.alive?/1` function by referencing its Pid.

```elixir
iex(2)> Process.alive?(my_process)
false
```

The process was spawned and immediately returned `:ok`. It had nothing else to do so it closed itself down; that is why the `Process.alive/1` returned `false`.

Notice, when we run iex it is actually a couple of processes as well. We can get the Pid of the iex session by calling `self/0`. `self/0` will always return the current process, think of it like `this`/`self` from an object language.


The Process Mailbox
-------------------
Every process has a mailbox, and we communicate with a process by sending it messages which will go to its mailbox. The process can choose to do something about these messages, or let them rot in the mailbox; this is done by either flushing the messages, using `flush/0` or with a `receive`-block.

```elixir
iex(3)> send self, :foo
:foo
iex(4)> send self, :bar
:bar
iex(5)> flush
:foo
:bar
:ok
iex(6)> flush
:ok
```

First we send ourselves the message `:foo`, and then the message `:bar` using the `send/2` function. Notice that the messages are stored until we do something about them; in this case we call the `flush/0` function, which is available in iex. Calling the `flush/0` function again will reveal that the messages has been removed from the mailbox.

The mailbox is a queue, processing its incoming messages on a first message in, first message out (FIFO) bases. When we want to react to incoming messages we use the `receive` construct.

```elixir
iex(7)> my_process = spawn(fn ->
...(7)>   receive do
...(7)>     :hi -> IO.puts "yo!"
...(7)>   end
...(7)> end)
#PID<0.105.0>
iex(8)> send my_process, :hi
yo!
:hi
```

This example uses a side effect to print `yo!` when we send the `:hi` message. The process actually does not know who send the message, just that it got the message `:hi`. If we would like to send the message back we could include the sender in a tuple.

```elixir
iex(9)> my_process = spawn(fn ->
...(9)>   receive do
...(9)>     {sender, :hi} -> send sender, :yo
...(9)>   end
...(9)> end)
#PID<0.112.0>
iex(10)> send my_process, {self, :hi}
{#PID<0.59.0>, :hi}
iex(11)> flush
:yo
:ok
```

Notice; we answer the sender by sending the message back using the same method that we used to send the message, and the message ended up in the mailbox of our iex-process.


Unknown message types
---------------------
In the previous examples we have set up processes that knew how to handle a message from a sender who said `:hi`. What would happen if the process received a message of a type it does not understand? Let us try that out:

```elixir
iex(12)> my_process = spawn(fn ->
...(12)>   receive do
...(12)>     {sender, :hi} -> send sender, :yo
...(12)>   end
...(12)> end)
#PID<0.120.0>
iex(13)> send my_process, {self, :how_do_you_do}
{#PID<0.59.0>, :how_do_you_do}
```

It just store the message in its mailbox. This is something we should have in mind, because a mailbox with a million messages lingering around could bring down a system. We can use `Process.info(my_process)[:messages]` to see debug information about the messages in our process. This is not something we would do in our code, because it is a heavy operation, but it is neat to do in a debugging session.

Notice that the `send/2` command evaluate to the message it is sending to the remote process. That is why iex will print the message to the console.


Linking processes
-----------------
When we spawn processes they are created without any relation to any processes. If something bad happens within the newly spawned process it will just die and no one will get notified about this:

```elixir
iex(14)> my_process = spawn(fn -> raise "oh my" end)
#PID<0.61.0>

22:53:32.824 [error] Error in process <0.61.0> with exit value: {#{'__exception__'=>true,'__struct__'=>'Elixir.RuntimeError',message=><<5 bytes>>},[{erlang,apply,2,[]}]}

iex(15)> Process.alive?(my_process)
false
```

Our newly spawned process died but it did not take the rest of the system down. This is a good thing, we are still in business, but most of the time we would like to have some kind of relation between processes: We can use `spawn_link/1` to create a two way connection between the process that create the process and the newly created process.

```elixir
iex(16)> spawn_link(fn -> raise "oh my!" end)
** (EXIT from #PID<0.59.0>) an exception was raised:
    ** (RuntimeError) oh my!
        :erlang.apply/2

22:57:34.582 [error] Error in process <0.66.0> with exit value: {#{'__exception__'=>true,'__struct__'=>'Elixir.RuntimeError',message=><<6 bytes>>},[{erlang,apply,2,[]}]}


Interactive Elixir (1.0.4) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)>
```

Notice that iex restarted; it's counter is back at one. We spawned and linked the failing process from iex which took down both processes. The iex prompt got restarted because it is supervised by another process, which promptly restarted the process when it got notified of its childs untimely demise.


Monitoring processes
--------------------
Tearing everything down when something fails is a strategy, but sometimes we want to just get notified that a process has died so that we can handle the situration accordingly, ie. respawn it without loosing our state. We can use `spawn_monitor/1` to create a monitored process.

```elixir
iex(2)> my_process = spawn_monitor(fn -> raise "oh my, again!" end)
{#PID<0.77.0>, #Reference<0.0.0.182>}

23:10:05.232 [error] Error in process <0.77.0> with exit value: {#{'__exception__'=>true,'__struct__'=>'Elixir.RuntimeError',message=><<13 bytes>>},[{erlang,apply,2,[]}]}
iex(3)>
```

Notice the prompt. The iex prompt was not torn down when our spawned process died. It got a notification about it though in its mailbox:

```elixir
iex(3)> flush
{:DOWN, #Reference<0.0.0.182>, :process, #PID<0.77.0>,
 {%RuntimeError{message: "oh my, again!"}, [{:erlang, :apply, 2, []}]}}
:ok
```

It recieved a `:DOWN`-message in its mailbox with information about which Pid failed and for what reason.


Trapping failures
-----------------
We now know: 1. how to spawn a process; 2. how to spawn a process that is linked to another process; and 3. how to monitor processes. We could build a tree of processes that rely on each other using these constructs. When we do so we are able to trap an error in our tree structure and by that way control which parts of the structure dies. We set a trap by setting a flag called `:trap_exit` to `true` in the process that we want to trap exits using `Process.flag/2`.

```elixir
iex(4)> Process.flag(:trap_exit, true)
false
my_process = spawn_link(fn -> raise "oh my, now that again?" end)
#PID<0.89.0>

23:28:24.339 [error] Error in process <0.89.0> with exit value: {#{'__exception__'=>true,'__struct__'=>'Elixir.RuntimeError',message=><<22 bytes>>},[{erlang,apply,2,[]}]}

iex(5)> 
```

Notice that we linked the process, and even though it was linked it did not kill the parent process. A message ended up in the mailbox though:

```elixir
iex(5) flush
{:EXIT, #PID<0.89.0>,
 {%RuntimeError{message: "oh my, now that again?"}, [{:erlang, :apply, 2, []}]}}
```


Working with modules
--------------------
So far we have seen functions being spawned, but it is possible to spawn a process from a module. The concepts are the same but the functions are: `spawn/3`, `spawn_link/3`, and `spawn_monitor/3`.

```elixir
spawn_link(MyModule, :my_function, [])
```

This would spawn and link a process using the function `my_function` on the `MyModue`-module with and empty list of arguments. The semantics are the same for the other variants.


OTP
---
Now we know that:

  * All processes has a mailbox
  * We can react on incoming messages in mailboxes using `receive`
  * We can have processes monitor other processes
  * We can link two processes making both go down if one should fail
  * We can trap exits, limiting the damage of a set of nodes that are going down

This is the stuff Supervisors and GenServers are made of.


### `GenServer`
Actually a GenServer is an abstraction that creates a `receive`-loop and passes on its state between receives:

  * `handle_call` is a message with the format: `{:$gen_call, *payload*}`
  * `handle_cast` is a message with the format: `{:$gen_cast, *payload*}`
  * `handle_info` is everything else.

That is why we would write a catch-all handler if we overwrite default `handle_info`. Otherwise our server could run out of memory because of processes with mailboxes full of unprocessed messages.


### `Supervisor`
A supervisor is a process that trap exits. It is responsible for starting and linking processes, which can be other supervisor processes, creating supervision trees.

Should a process die it will get respawned using a given strategy: `:one_for_one`, `:one_for_all`, `:rest_for_one`, or `:simple_one_for_one`.

Should a given supervisor get terminated it will take all its children with it.

Restarting children should be planned carefully, because the new child will be a new process: Every process that need to know about the process need to know that it now have new and different Pid, and the state which it is initialized will have to be calculated somehow, or fetched from a database.

Also, great care should be taken when saving state, as trying to respawn a failing process from corrupt state will lead to nowhere fast. This is tricky stuff, but it is true for any programming paradigm out there--we deal with it by having options for how fast to retry failing respawns, and options for how many respawns we will attempt, before the supervisor finally kills itself and throw the error to its supervisor, who will respawn that branch according to its specifications--or take down the entire system if it was top level--game over.

todo, perhaps: mention `worker` and `supervise`


Conclusion
----------
Processes are an essential part of programming for the BEAM. Elixir is a great, fun, and potentially productivity enhancing programming language; even when we do stuff that does not use them--but the language and its eco-system really shines when we bring processes to the mix, so we would do ourselves a disservice if we do not get familiar and comfortable with them.

It is good to know how Supervisors and GenServers are constructed, and we should go out of our ways to put stuff into GenServers and Supervisors because these are battle tested and hardened way better than our own attempts on building these constructs could ever be, but sometimes we may have to roll our own.
