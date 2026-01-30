I/O Queue Pair

An I/O QP consists of one SQ and its corresponding CQ, which are used to perform data transfers (I/O operations). Multiple I/O QPs can be created to enable parallel I/O operations, allowing each QP to function independently and maximizing the use of multi-core processors.

  

I/O SQ – the queue where the host places read, write, and flush commands

I/O CQ – the queue where the controller posts completion entries after processing the commands