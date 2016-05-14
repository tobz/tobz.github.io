---
layout: post
title: "Adventures in compression: Huffman coding"
description: ""
---
{% include JB/setup %}

Rather than start with compression algorithms as you might know them -- DEFLATE, GZip, etc -- we're going to start with one of the underlying components used in DEFLATE compression, called __Huffman coding__.

Why are we starting with Huffman coding?  Well, in a DEFLATE-compressed output stream, all of the compression is indicated by Huffman coding.  We can fundamentaly understand what's going on, but Huffman coding is how it's all put together, so we'll start here first.

## What's in a coding, anyways?

As alluded to in the [prelude]({% post_url 2016-05-06-adventures-in-compression-prelude %}), compression is, at its core, coming up with a representation for a piece of content that is smaller than what you started with, and in a way that can be reversed after-the-fact.  __Huffman coding__ is a play on this technique that uses variable-length coding of a set of source symbols, where a symbol (or byte, if that's easier to grok) is mapped to by a variable-length code.  In practice, you can think of this as:

    A -> 01 (2-bit code)
    B -> 101 (3-bit code)
    C -> 0010 (4-bit code)

Which is to say, in your compressed output, seeing the bits __01__ means you can replace those two bits with an __A__, and so on.  _But binary is all zeroes and ones, how do you keep track of that?!_  You're right.  We need a way to make sure that as we're reading this bitstream, we don't confuse some bits with the code for another symbol, which brings up to another point about Huffman coding... it's a _prefix-free_ code system.

This means that a code generated for a symbol will never be the prefix of another symbol's code.  This is important because, as mentioned above, we wouldn't want to confuse one code with another, and since Huffman coding is variable-length, the code for one symbol could be shorter than the code for another symbol, which otherwise could lead to the smaller code being a subsequence of a larger code, making this scheme unworkable!

<!--more-->

## How the hell do we use it?

While we can describe what Huffman coding is, we have to talk about how it's used, otherwise we'll never understand how it obtains the properties that it does.

Huffman coding happens by constructing a _Huffman tree_.  A Huffman tree is a binary tree, meaning every node has, at most, two children, or branches.  A leaf node (a node that has no branches) in a Huffman tree represents a symbol and its _weight_.  Since Huffman trees usually represent the symbols in a given piece of content, the weight is often equivalent to the frequency with which the symbol occurs.  Any node that isn't a leaf node has a weight that is equal to the weight of its children.  In constructing the tree, the symbols with the lowest weight are placed at the bottom, so that the root node of the tree is alway the greatest weight.

As we'll see later on, the algorithm to construct the Huffman tree is what implicitly gives more-frequent symbols shorter codes -- saving space when compressed -- and also ensures that one symbol's code will never be contained within the code for another symbol.  These are the "variable-length" and "prefix-free" properties we mentioned before.

### Figuring out our corpus

Let's pretend we're compressing a random English idiom:

    it takes two to tango

Now, let's count the frequency with which each character (or symbol) appears.  We'll refer to the frequency as the "weight" of the symbol.  While doing so, we'll also sort the symbol/weight pairs so that the symbols with the lowest weight are first, or on top.  When we're done, we're left with this:

     symbol |   weight
    --------------------
       e          1
       g          1
       k          1
       n          1
       s          1
       w          1
       a          2
       o          3
      ' '         4
       t          6

You can use whatever data structure you like to tally up the weight of symbols.  In order to build the tree, we'll need to use a queue, and that queue will need to keep all items sorted, so we'll need a priority queue.  If there is an alternative data structure that fulfills this purpose, you can use that, but we'll be using terminology related to queues: enqueue and dequeue.

### Building the tree

Since we're building a tree, we'll be working with nodes.  These nodes will have a field for the symbol and the weight, in addition to two fields for a left and right child.  Before starting, take all the symbol/weight pairs and convert them into nodes.  Your priority queue should be configured to have the lowest weight be the highest priority i.e. the first item you get when dequeuing should be the lowest weight.

Once the priority queue is populated, we can begin constructing the tree.  The process goes as follows:

    - pop two nodes off the queue (these will be the two lowest-weight items)
    - create a new node, with both leaf nodes as its children
    - set the weight of the new node as the sum of its children's weight
    - push that new node into the queue
    - go back to step 1 and repeat until there is only one node left

When we have only one node left, that is the root of our Huffman tree, and the tree is complete.  Let's build this by hand to demonstrate the process.  For the nodes that don't represent symbols, their "symbol" will simply be an asterisk.  The nodes are displayed as __symbol::weight__.

<div class="text-center" markdown="1">
![showing the steps of building a Huffman tree](/assets/posts/huffman-coding/tree-building.gif)
</div>

Wooo, we now have a tree... but what the heck do we do with *that*?  We need one more bit of understanding before we can start using our tree: the codes!

### Climbing all over

The fact that a Huffman tree is a binary tree, and that it uses binary codes, is not a coincidence.  If we consider a node in a binary tree, with two potential children, there are only two directions you can go from that node -- left or right.  Since binary is a base 2 numeric system -- 0 and 1 -- a single bit can represent which direction to move.  Let's explore what a Huffman tree looks like if we assign bit values to each node.  If a node is the left child of a parent node, we'll give it a zero.  If it's the right right, we'll give a one.  Our root node will have no value, since it has no parent.

<div class="text-center" markdown="1">
![showing the tree with zeroes and ones assigned to nodes](/assets/posts/huffman-coding/tree-with-zeroes-and-ones.png)
</div>

Now that we've assigned a bit value to each node in our tree, let's consider for a moment what this buys us.

Any symbol which falls on the left side of the root node will always start with a zero bit.  Similarly, any symbol which falls on the right side of the root node will always start with a one bit.  If we go down a level where we have the __*::8__ and __*::13__ nodes, all the symbols on the left side of the __*::8__ node will always start with the bits "00", and all the symbols on the right side will start with the bits "01".  If we repeat this exercise, going all the way to the leaf nodes, what we're actually able to show is that every branch is unique.  There is never a way for something on the right side of the root node, for example, to ever start with a zero bit.  That notion is applied at every level, so that when we reach a symbol, the bits of the nodes which proceed it are always guarenteed to be unique that to symbol.  If that symbol shares a sibling on the other side, they both will have that unique prefix, but they won't have the same code... because the child on the left will always have a zero bit, and the child on the right will always have a one bit.  This is how Huffman coding guarentees that all codes are _prefix-free_.


We've now assigned a code to each node in our tree.  Now we can generate an actual code for each _symbol_ in the tree.  A potential approach here is to do a depth-first search, and to pass along a stack/array of the codes each node you've encountered along the way had.  For every node (except the root node, because it's our starting point) you take its code and add it to this stack/array.  When you detect that you've hit a leaf node -- aka a node with a symbol -- you take the current state of the stack (including the code for that node itself) and that is the code the symbol gets.

Now that we understand how these bit values apply to each node in the tree, let's use them to actually figure out what the code would be for a given symbol.  Every time we reach a new node with a bit value, we'll add it to a stack.  This could be an array or a string, but what we're trying to get at here is that we're always appending the bit value to a container, so that when we reach a node which has a symbol, we know the "path", or the bit values, that it took to get there.  We'll use __k__ as our symbol for this example:

    - start at root node (stack: empty)
    - go down left branch to *::8
    - code value is 0, add to stack (stack: 0)
    - go down left branch (ignoring this for now)
    - go down right branch to *::4
    - code value is 1, add to stack (stack: 01)
    - go down left branch to *::2
    - code value is 0, add to stack (stack: 010)
    - go down left branch to k::1
    - code value is 0, add to stack (stack: 0100)
    - node is leaf node, return and add mapping for "k" to "0100"

If we apply this process to the whole tree, we'll end up with a table like this:

     symbol  |   code
    --------------------
      ' '         00
       t          10
       o          110
       a          1110
       k          0100
       n          0101
       s          0110
       w          0111
       e          11110
       g          11111

We now have codes for all of our symbols!  An important note here:

I formulated the original Huffman tree by hand, with some cherrypicking of how I selected values, in an attempt to generate a pleasant-looking tree.  The tree is valid, as far as I can tell, but based on the implementation of your priority queue, and how it handles sorting, your tree might end up slightly different.  In turn, this means you could get slightly different, possibly more optimal, codes for the same example text.

Moving on, let's add in the frequency of each symbol next to the code generated for it for fun:

     symbol  |  frequency  |  code
    ---------------------------------
      ' '           4          00
       t            6          10
       o            3          110
       a            2          1110
       k            1          0100
       n            1          0101
       s            1          0110
       w            1          0111
       e            1          11110
       g            1          11111

Now it's trivial to see: the higher weight, or frequency, a character, the shorter of a code it has.  This is part of the magic: if we replace the things that occur more frequently with shorter replacements, we'll save more space.  We get this optimization "for free" because of how the algorithm to build the tree works.  Cool stuff. :)

### Back and forth, to and fro

We've built our Huffman tree and we've figured out how to generate the codes for our symbols with it.  Now we can compress and decompress our example string.

Compressing is a piece of cake: symbol by symbol, find the code for the symbol, and write it out as bits.  In practice, you're not just going to write bits to an output stream.  You'll have to write bytes.  Most implementations of Huffman encoding I've found will do something like: create a small 4-byte buffer, and do bit shifting to simulate writing bits, and when the buffer overflows, it writes those 4 bytes, or however many, to some actual output stream, and reinitializes the small 4-byte buffer. (TODO what happens if you're not on a byte boundary? special end of file symbol type thing?)

Decompressing involves traversing the tree.  Imagine, using our example tree, you encounter the bits "0101".  Take that very first bit, 0, and go to the left side of the root node.  Now, take the second bit, 1, and take the right path.  You keep repeating this process, reading one bit at a time, until you hit a leaf node.  Once you hit that leaf node, you output the respective symbol to your "decoded" stream and you reset yourself, starting at the root node again.  You repeat this until you've hit the end of your stream, and now you have your decompressed text.  Ultimately, you're just following the bits down the left of right side of a node until you hit a symbol, as we mentioned previously. (TODO same as above, how do we know when the stream has ended?)

### Wrapping up

We've now learned that Huffman coding is a _variable-length_, _prefix-free_ code system, which represents input symbols (commonly bytes, or other primitives like a short or an int) and generates a variable-length binary code based on the weight, or frequency, of which the symbol occurs in some known input.  This code system generates variable-length codes, which can lead to saving multiple bits per input symbol.  It also generates prefix-free codes, so that one symbol's code is never the prefix of another symbol's code.

We've also learned that Huffman code systems are represented by Huffman trees, a binary tree constructed in such a way that the codes generated for symbols are guarenteed to be prefix-free.  This tree can be used to generate a mapping of symbol to code, allowing easy compression.  It can be traversed, reading back the bits of the encoded one by one, using each bit as a decision on whether to take the left or right branch of a given node in the tree.

### What's next?

With a basic understanding of Huffman coding under our belts, I plan to explore some related items.  Potential topics include other coding systems -- Shannon-Fano coding, for example -- and other tweaks on Huffman coding.  Going through the basics to talk about Huffman coding, I really enjoyed the simplicity of it, and I'm going to be looking to write about other such systems and algorithms before we jump right into "real" compression algorithms like DEFLATE, etc.
