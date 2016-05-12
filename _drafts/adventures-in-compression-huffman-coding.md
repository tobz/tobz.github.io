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

Alright, we now know that Huffman coding is a _prefix-free_, _variable-length_ code system that maps symbols to codes.  Well, we need to actually map symbols to codes.  How do we do that?  First, we need to remind ourselves what we're after: compression.  Compression operates on replacing common pieces of content with a smaller representation.  Repetition is highly compressible.  Huffman coding operates at the symbol level, which isn't to say it couldn't represent chunks of text, but it's commonly used to represent bytes.

Thus, the common technique is to establish the frequency is which a given symbol (byte) occurs in our content.  As we construct the Huffman tree (the representation for a Huffman code system against a particular corpus), the symbols with the higher frequency, or weight, will be given shorter codes.  Things that occur more often get shorter codes, which effectively compresses them.  Sweet!

### Figuring out our corpus

Let's pretend we're compressing a random English idiom:

    it takes two to tango

Now, let's count the frequency in which each character appears, leaving us with a table like this:

     symbol | frequency
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

We've specifically sorted the table with characters containing the lowest frequency first.  No need to go further here with then sorting symbols lexicographically or anything. In practice, you would probably accomplish this with a priority queue, or you could shove all the tuples of (symbol, frequency) into an array and sort them there.  Either way, we'll be using terminology that relates to a queue -- push, pop -- as we talk about interacting with this list.

### Building the tree

The process goes roughly as follows:

    - pop two items off the queue (this will be the two lowest-frequency items)
    - create a leaf node for each of them (store the symbol and frequency)
    - create an branch node, with both leaf nodes as its children
    - set the frequency of the branch node as the sum of its children
    - push that branch node into the queue
    - go back to step 1 and repeat until there is only one node left

When we have one node, that is the root of our Huffman tree.  Let's build this by hand to demonstrate the process.  For our leaf nodes, we'll represent them as __value::frequency__, and we'll represent intermediate nodes as __*::frequency__:

![showing the steps of building a Huffman tree](/assets/posts/huffman-coding/tree-building.gif)

Wooo, we now have a tree... but what the heck do we do with *that*?  We need one more bit of understanding before we can start using our tree: the codes!

### Climbing all over

We need to take our tree and use it to generate codes for all of the different symbols.  To do so, we'll recurse through our tree and assign each node either a zero or a one.  We'll this value the node's "code".  Starting at the root node of the tree, give the child node on the left a code of zero, and the child node on the right a code of one.  If either of those child nodes have their own child nodes, do the same thing.  Repeat the process until you run out of child nodes.  It should end up looking like this:

![showing the tree with zeroes and ones assigned to nodes](/assets/posts/huffman-coding/tree-with-zeroes-and-ones.png)

We've now assigned a code to each node in our tree.  Now we can generate an actual code for each _symbol_ in the tree.  A potential approach here is to do a depth-first search, and to pass along a stack/array of the codes each node you've encountered along the way had.  For every node (except the root node, because it's our starting point) you take its code and add it to this stack/array.  When you detect that you've hit a leaf node -- aka a node with a symbol -- you take the current state of the stack (including the code for that node itself) and that is the code the symbol gets.

Based on our example tree, let's go through the operations that would map the bottom left-most character, __k__:

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

Now it's trivial to see: the higher frequency a character, the smaller of a code it has.  This is part of the magic: if we replace the things that occur more frequently with shorter replacements, we'll save more space.  We get this optimization "for free" because of how the algorithm to build the tree works.

### Back and forth, to and fro

So, now that we have the tree and our mapping, we can use them to encode or decode the corpus!

With our original example text now fully represented in our Huffman tree, and having codes generated for all the symbols, we're able to compress and decompress the example text!

Compressing is a piece of cake: symbol by symbol, find the code for the symbol, and write it out as bits.  In practice, you're not just going to write bits to an output stream.  You'll have to write bytes.  Most implementations of Huffman encoding I've found will do something like: create a small 4-byte buffer, and do bit shifting to simulate writing bits, and when the buffer overflows, it writes those 4 bytes, or however many, to some actual output stream, and reinitializes the small 4-byte buffer. (TODO what happens if you're not on a byte boundary? special end of file symbol type thing?)

Decompressing involves traversing the tree.  Imagine, using our example tree, you encounter the bits "0101".  Take that very first bit, 0, and go to the left side of the root node.  Now, take the second bit, 1, and take the right path.  You keep repeating this process, reading one bit at a time, until you hit a leaf node.  Once you hit that leaf node, you output the respective symbol to your "decoded" stream and you reset yourself, starting at the root node again.  You repeat this until you've hit the end of your stream, and now you have your decompressed text.  Ultimately, you're just following the bits down the left of right side of a node until you hit a symbol. (TODO same as above, how do we know when the stream has ended?)
