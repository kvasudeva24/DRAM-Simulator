I have just learned about the DRAM heirarchy in my Computer Systems Class and now want to implement how DRAM works from the memory controller all the way to the matrix level in Banks. 
I think this will help me write OOP code as well as know how systems work. This should be fun. I will update the read me on how I want to design the simulator before I start coding.


Order of a DRAM controller 

Memory Controller (1...*)
channel (1)
DIMMS (1..*)
Rank (0..1)
Banks (1...*)
Subarray (1...*)
Matrix (1...*)

This is the conceptual organization of the heirarchy of DRAM.
Memory controller has a memory channel to multiple DIMMS
Which have 2 ranks
Each rank has multiple banks
Each bank has multiple subarrays 
The subarrays hold the actual data

This the conceptual implementation of the heirarchy to DRAM. In practice we have DRAM chips that are lined up side by side. x8 means you have 8 data pins per DRAM chip
and each DRAM chip is responsible for sending 8bytes or 64 bits. Since we define our cache block size of the LLC to be 64 bytes we move that much out of DRAM (512 bits) at a time

x8 DRAM chips are also very common

I will try to build my knowledge of the DRAM system from the bottom up as I hope it will help me later on when I code my implementation.


First I want to address that on modern machines there are multiple memory controllers to which the LLC will route date too. 
This is to support multiple accesses to DRAM in parallel. It is usually blocked off in a way such that eteh first 64 bytes go through channel A
and the next channel B and so on and so forth.

I also understand to get the maximum throughput from the hardware we will stripe the data across a chip such that when we send the 64bytes, 
we will be able to data burst.

To begin, I will start with my design specefications:










