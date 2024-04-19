---
layout: page
title: "Moderately Hard Functions"
permalink: /projects/mhfs
---

### About ###

This project came about during the summer 2022, during which Professor Leo Reyzin introduced me to this intriguing, tough problem at the intersections of complexity theory, cryptography, engineering, and physics. I've tried to include a brief (hopefully accessible) introduction to the problem for anyone curious.

### Mini-summary ###
Traditionally, a cryptographer's main goal is to design clever methods of communication while in the presence of bad actors. Maybe you've heard of secret codes that allow us to send letters to people halfway across the world, while ensuring only the recipient can read it. Perhaps you've heard of encryption machines used during the war to encode secret military communications. In all of these cases, the information being communication is being protected from some (potentially unknown) adversary.

The natural challenge with hiding information is that we have absolutely no clue who we're hiding it from and what they're capable of; are we fighting against a naive toddler or a super genius? Does the adversary have a computer of some kind, or just a pen and pencil? If a computer, which kind? We have no clue. But it's only right we protect against the worst case scenario. So cryptographers always assume we're dealing with some super genius that may hold knowledge of problems we don't even know how to solve. In some sense, we assume the adversary is arbitrarily strong.

Cryptographers have come up with many neat tricks to formalize what this means. Of course, there is nuance here, but the main point is that we're able to design online systems that take an unreasonable amount of time/resources for adversaries of arbitrary strength to break, while being easy for the recipient to participate and access the message. And thus, we say these systems are secure (against some adversary we define). Phew...

Now that we have a loose idea of how cryptographers formulate and think about an imaginary adversary, I want to slowly introduce the idea of egalitarian computing. In other words, let's think deeply about the asymmetry between an attacker and a recipient within a cryptographic scheme. While we want to protect against an arbitrarily strong attacker, we want to make sure the recipient can still efficiently access the message. This is the sort of 'trapdoor' of resources that appears frequently within cryptography (easy to compute in one direction, but hard in the other).

Put simply, assumed attackers and recipients have an asymmetry in resources. We assume attackers are all-powerful and assume recipients are weak (both are worst case scenarios). And in the real world, this appears frequently: while your phone doesn't expand a ton of computational resources to submit your password to a website once, hackers may use hundreds of powerful machines to guess your password.

As you let that marinate, you're probably asking yourself: well, what does it mean to be cheaper or easier to compute something? Well, computer scientists usually formulate the how easy or hard a program or computation is with some sort of complexity measure. You may have heard of Big-O, which considers asymptotic runtime of an algorithm. Maybe you've even heard of the memory complexity of an algorithm, or how much memory it uses as the input scales. While I won't go deep into complexity, it suffices to say that computer scientists have a few different ways to formalize the hardness of an algorithm. It allows them to compare programs and decide which are better in an objective manner and tailored to a specific setting.

Stepping away from our attacker/recipient paradigm, I'd like to propose a slightly modified setting. Imagine, ignoring what we discussed thus far, we have a function or algorithm that is easy to compute, but hard (for our purposes near impossible) to invert. In other words, it's easy to compute an output given an input, but is infeasible to compute an input for a given output. Here, easier and harder can be thought of as some sort of complexity measure.

Such a function, is actually incredibly powerful in many cryptographic settings. Let's call this proposed function a trapdoor function. Perhaps most commonly use for it is storing passwords. We can store a trapdoor output for a given password input and not have to worry about anyone who steals this value from learning your secret password (it is hard to invert the trapdoor function).

If we weaken this trapdoor so that it's feasible to break, but takes a lot of resources, we can call this a moderately hard function. Moderately hard functions are very powerful within many settings where we wish to establish relatively large resource requirements and meter access to a resource. Protecting against spam is one common application. Within the consensus mechanisms of Bitcoin, moderately hard functions are used to make sure a certain amount of work or effort was expanded by a participant. This ensures that only an unreasonably large, expensive, and coordinated attack could ever take down the system.

But with the economic incentives of industries like Bitcoin mining, many have found it profitable to construct specialized computing chips and machines that are able to decrease their costs of computing such moderately hard functions. You may have heard of big crypto-currency mining rigs built in countries like Greenland that expand massive amounts of energy to make money off the Bitcoin network. These rigs use special Application Specific Integrated Circuits (ASICs) to effectively decrease the energy and time cost of computing these moderately hard functions.

Hopefully you can sort of see the issue here: we essentially lose the egalitarian nature of our moderately hard computation. Over time, security of Proof of Work consensus (the specific approach used by Bitcoin) would break as mining power gradually concentrates in the hands of these wealthy miners. Furthermore, protecting and securing data becomes more difficult when ASICs can effectively equalize trapdoor inversion.

While I've left out many details and simplified heavily, I hope you've got a taste of the interesting questions that lie here. Most notably, how do we even begin to formalize a measure of hardness that is compatible with a truly egalitarian function? It seems one isn't possible? Yet, borrowing various complexity measures has proven to be fruitful in the past; researchers have looked to the memory costs of integrated circuits, as well as the bandwidth costs involved at the cache level. Perhaps there is something useful lying within the organization of circuits themselves that can unlock a certain hardness measure that we can't easily optimize for.

Some useful links to read more:
* [Moderately Hard Functions: Definition,  Instantiations,  and Applications](http://dx.doi.org/10.1007/978-3-319-70500-2_17)
* [Bandwidth Hard Functions for ASIC Resistance](https://eprint.iacr.org/2017/225.pdf)
* [$$\mathtt{Scrypt}$$ is Maximally Memory-Hard](https://eprint.iacr.org/2016/989.pdf)