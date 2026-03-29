+++
date = '2021-12-11T15:30:48-04:00'
title = 'My Thoughts About Java Reactive Programming'
+++

# Context

Traditional Java applications use thread pools for simultaneous I/O operations (such as a REST call). Each request consumes a thread freed at the end of the processing only. So, whenever the thread pool is empty, new requests are blocked waiting for an available thread. This programming paradigm is called imperative or blocking.

# What is reactive programming?

It is a programming paradigm based on the data transmission from one or more sources called Publishers to other elements called Subscribers in an asynchronous, non-blocking, and functional way. Streams combined with the Observable design pattern process all types of data.

# How to use it?

RxJava or Project Reactor - used in Spring WebFlux - are two common choices to implement Reactive programming.

# Pros

*   Precise, concise, and cleaner code
    
*   Non-blocking asynchronous requests
    
*   Mainly increases resource usage on multi-core and multi-CPU hardware
    
*   Increase developer productivity
    
*   Reactive systems come with improved scalability and resilience
    

# Cons

*   Uses more memory to store data streams most of the time
    
*   Increased cost of debugging
    
*   Steep learning curve
    
*   Lack of quality and easy-to-read resources
    
*   Often confused to be equivalent to functional reactive programming
    

# Conclusion

I think the reactive paradigm is a great choice when the application satisfies the following two conditions:

1.  reactive libraries are available for all the external services called
    
2.  must be able to support a high number of simultaneous connections
