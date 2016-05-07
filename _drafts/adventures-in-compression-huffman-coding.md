---
layout: post
title: "Adventures in compression: Huffman coding"
description: ""
---
{% include JB/setup %}

Rather than start with compression algorithms as you might know them -- DEFLATE, GZip, etc -- we're going to start with one of the underlying components used in DEFLATE compression, called __Huffman coding__.

Why are we starting with Huffman coding?  Well, in a DEFLATE-compressed output stream, all of the compression is indicated by Huffman coding.  We can fundamentaly understand what's going on, but Huffman coding is how it's all put together, so we'll start here first.

### What's in a coding, anyways?

As alluded to in the [prelude]({% post_url 2016-05-06-adventures-in-compression-prelude %}), compression is, at its core, coming up with a representation for a piece of content that is smaller than what you started with, and in a way that can be reversed after-the-fact.  __Huffman coding__ is a play on this technique that uses variable-length coding of a set of source symbols, where a symbol (or byte, if that's easier to grok) is mapped to by a variable-length code.  In practice, you can think of this as:

    A -> 01 (2-bit code)
    B -> 101 (3-bit code)
    C -> 0010 (4-bit code)

Which is to say, in your compressed output, seeing the bits __01__ means you can replace those two bits with an __A__, and so on.  _But binary is all zeroes and ones, how do you keep track of that?!_  You're right.  We need a way to make sure that as we're reading this bitstream, we don't confuse some bits with the code for another symbol, which brings up to another point about Huffman coding... it's a _prefix-free_ code system.

This means that a code generated for a symbol will never be the prefix of another symbol's code.  This is important because, as mentioned above, we wouldn't want to confuse one code with another, and since Huffman coding is variable-length, the code for one symbol could be shorter than the code for another symbol, which otherwise could lead to the smaller code being a subsequence of a larger code, making this scheme unworkable!

<!--more-->

### How the hell do we use it?

Alright, we now know that Huffman coding is a _prefix-free_, _variable-length_ code system that maps symbols to codes.  Well, we need to actually map symbols to codes.  How do we do that?  First, we need to remind ourselves what we're after: compression.  Compression operates on replacing common pieces of content with a smaller representation.  Repetition is highly compressible.  Huffman coding operates at the symbol level, which isn't to say it couldn't represent chunks, but it's commonly used to represent bytes.

Thus, the common technique is to establish the frequency is which a given symbol (byte) occurs in our content.  As we construct the Huffman tree (the representation for a Huffman code system against a particular corpus), the symbols with the higher frequency, or weight, will be given shorter codes.  Things that occur more often get shorter codes, which effectively compresses them.  Sweet!

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

Now, let's build our tree!  The process goes roughly as follows:

    - pop two items off the queue (this will be the two lowest-frequency items)
    - create a leaf node for each of them (store the symbol and frequency)
    - create an branch node, with both leaf nodes as its children
    - set the frequency of the branch node as the sum of its children
    - push that branch node into the queue
    - go back to step 1 and repeat until there is only one node left

When we have one node, that is the root of our Huffman tree.  Let's build this by hand to demonstrate the process.  For our leaf nodes, we'll represent them as __value::frequency__, and we'll represent intermediate nodes as __*::frequency__:

![showing the steps of building a Huffman tree](/assets/posts/huffman-coding/tree-building.gif)
