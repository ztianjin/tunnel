# Tunnel

Tunnel is a data driven framework for distributed computing in Torch 7.

## Why data driven?

Machine learning systems are intrinsically data-driven. There is data everywhere -- including training and testing samples, the model parameters, and various state used in different algorithms. This means that by carefully designing and implementing various synchronized data structures, we can make it possible to program distributed machine learning systems without a single line of synchronization code. It is in contrast with program-driven methodology where the programmer has to take care of mutexes, condition variables, semaphores and message passing himself.

## Installation

The following prerequisites must be installed:
* [Torch 7](https://github.com/torch/torch7)
* [threads](https://github.com/torch/threads)
* [tds](https://github.com/torch/tds)

Then, you can just copy the `tunnel` directory to your Lua installation directory, such as `/usr/share/lua/5.1/tunnel`.
```bash
$ git clone git@github.com:zhangxiangxiao/tunnel.git
$ sudo cp -r tunnel/tunnel /usr/share/lua/5.1/
```

A luarocks specification will be added later when this repository is tested further.

## Example: Consumer-Producer Problem

Here is an example that demonstrates how to write a [producer-consumer problem](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem) solver without a single line of synchronization code using tunnel.
```lua
local tunnel = require('tunnel')

-- This function produces 6 items and put it in vector.
local function produce(vector, atomic)
   for i = 1, 6 do
      local product = __threadid * 1000 + i
      vector:pushBack(product)
      atomic:write(function() print('produce', __threadid, i, product) end)
   end
end

-- This function takes 4 items from vector and takes 1 second to consume each.
local function consume(vector, atomic)
   local ffi = require('ffi')
   ffi.cdef('unsigned int sleep(unsigned int seconds);')
   for i = 1, 4 do
      local product = vector:popFront()
      atomic:write(function() print('consume', __threadid, i, product) end)
      -- Pretend it takes 1 second to consume a product
      ffi.C.sleep(1)
   end
end

local function main()
   -- Create a syncronized vector with size hint 4
   local vector = tunnel.Vector(4)
   -- Create a dummy atomic as a synchronization guard for prints
   local atomic = tunnel.Atomic()

   -- Create a 2 producer threads and 3 consumer threads
   -- In total they produce 12 products, and consume 12 products.
   local producer_block = tunnel.Block(2)
   local consumer_block = tunnel.Block(3)

   -- Link vector and atomic with both producer and consumer blocks
   producer_block:add(vector, atomic)
   consumer_block:add(vector, atomic)

   -- Execute the producer and consumer threads
   producer_block:run(produce)
   consumer_block:run(consume)
end

-- Call the main function
return main()
```

One possible output:
```
$ th producer_consumer.lua
produce 1       1       1001
produce 1       2       1002
produce 1       3       1003
produce 1       4       1004
consume 1       1       1001
produce 1       5       1005
consume 2       1       1002
produce 2       1       2001
consume 3       1       1003
produce 1       6       1006
consume 1       2       1004
produce 2       2       2002
consume 2       2       1005
produce 2       3       2003
consume 3       2       2001
produce 2       4       2004
consume 1       3       1006
produce 2       5       2005
produce 2       6       2006
consume 2       3       2002
consume 3       3       2003
consume 1       4       2004
consume 2       4       2005
consume 3       4       2006
```

### How Does the Example Work?

First of all, the `main` function created two synchronized data structures -- one is `vector` of type `tunnel.Vector` and the other `atomic` of type `tunnel.Atomic`. It then created two thread blocks representing 2 producer threads and 3 consumer threads, and add the data structures `vector` and `atomic` to the blocks. For these thread blocks, when we call `block:run(callback)`, the `callback` function will obtain these data structures in order and be able to use them. The producer block runs the `produce` function, and the consumer block runs the `consume` function.

When a thread in the producer block runs the `produce` function, it obtained the `vector` and `atomic` data structures. After it produced a product, it will attempt to put it in `vector` by calling `pushBack`. However, when it will make the size of `vector` larger than 4 (a size hint we used to initialize `vector` in `main` function), it will wait until a consumer has removed a product before `pushBack` returns. Then, it calls `atomic:write` with a callback to print information synchronously. It can do so because the type `tunnel.Atomic` is essentially a reader-writer wrapper, in which when `atomic:write(callback)` executes `callback` it can be sure that only one thread (even in different blocks) is accessing data, providing an exclusive guard for `print` in this case.

The consumer thread simply pop from `vector` by calling `popFront()` and print information in the same way using the same `atomic` guard. It then pretends that it takes 1 second to consume the product before going to the next iteration. The call `vector:popFront()` is also synchronized, in the sense that when `vector` is empty it will wait untill a product is available then return.

The example simply demonstrates usefulness of synchronized data structures and data-driven programming.
