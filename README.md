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

Furthermore, in the hardware there is a subarray abstraction to make sure the capacitance is passed onto the sense amplifier. This is not true 
in the memory controller. This is a level of abstraction.

To begin, I will start with my design specefications:

I will have 32 bit addressing support. Thus that means I can serve 4GB of memory. We will have two memory channels meaning that each channel can service 2 GB.

2 DIMMS = 1GB per Dimm
2 ranks per DIMM = 512MB per rank

We can design our system for 16 banks which leaves us with 
32MB per bank

Banks logically span the entire width of the Rank on the DIMM chip
If we wanted to think of a bank as large square, we need 25 bits to index 32MB. Whether that be in some sort of square or rectangle. However
since we need a 64 byte cache line we only need 19 bits as 2^6 = 64. and we dont need to index every possible byte. 

19 row/column + 4 bank + 1 rank + 1 DIMM + 1 bit to decide which memory controller to go (determined by LLC) + 6 bits for cache line offset

This is the abstraction provided by DRAM to the MEM controller. However we cannot have a 2^10 x 2^9 square bitline into a sense amplifier or else the capacitance will be lost. we need to make it smaller. 

Since we have 19 bits to play with we can have 9 bits used for the subarray inside a bank. this means we have 512 subarrays. then our matrix can be a 2^5 by 2^5 grid. this is an 1024 byte matrix and we have 512 of them. 

A single Bank is composed of 512 Subarrays. This ensures that bitline lengths remain short enough to satisfy the electrical requirements of the sense amplifiers, a detail abstracted away from the Memory Controller but vital for cycle-accurate latency modeling.

Here is the address breakdown for the memory controller:

0-5: offset
6: memory controller interleave
7: DIMM
8: Rank
9-12: Bank
13-31: row/column ( we want the subarray to change the least so inside the bank,
                    13-17: column, 18-22 row, 23-31: subarray)
                









