---
layout: post
title: "Erlang Real Time Server - Introduction"
date: 2012-05-11 19:53
comments: true
categories: Erlang Diameter
---

Erlang RTS - Introduction

What I am planning to do is a 'pet project' utilising my new passion - functional programing in Erlang. In this project I'll sketch some ideas for a telecom "Online Charging System"(OCS). I have no intention to implement a real OCS as defined in the 3GPP specifications. It will be just a small part of what is more popular in non-3GPP world as a "Real Time Server" (RTS). 

"WTF is a RTS?" - you are asking. Well, when it comes to the Billing System domain, this is a piece of software, which is responsible for the Authentication, Authorisation and Accounting (AAA) services. This is the part of the system which grants you access to and charges for the usage of a given service, such as - network access, voice calls, video on demand, text messages, etc. 
  
The basic parts of the usual RTS are: 

* AAA protocol server, which will receive the AAA requests.
* A module, I will call it "Aggregation module", which purpose is to calculate the answers (for those of you familiar with the 3GPP - it is somewat combined Account Balance Management Function (ABMF) and Rating Function (RF) ) .

The AAA protocol is an important part. If you have ever heard about 'RADIUS' - that's it. It is a protocol widely used in ISPs and VoIP operators. What I am planning to use, however, is the RADIUS successor - Diameter. Partially, because the standard Erlang distribution has a Diameter library and partially, because I want to learn more about Diameter. 

The second part, which is even more interesting - the Aggregation - is the place where all the calculations will be done. For example - if you want to watch a film on demand or make a voice/video call the system has to check whether you have enough money on your balance and will grant or not access to the service - that's the 'authorization' part. After the end of the call it will calculate the amount to deduct from your balance, depending on the time you spent talking and the destination (local or international call, etc.). That's the 'accounting' part. Since there can be between hunderds and millions (if you are someone like AT&T :-)  ) of subscribers, simultaneously using your services, the system has to be scalable. By 'scalable' I mean - painlessly adding a new Aggregation module with a minimum effort. It looks like Riak would fit this purpose perfectly.

A note to the readers - this is not a tutorial for beginners and some initial knowledge in Erlang will be required. I highly recommend Frederic Trottier-Hebert's [Learn You Some Erlang](http://learnyousomeerlang.com). Also, as my expirience in Erlang, Diameter and Riak is relatively small, I strongly recommend any of the following publications to be used only for educational purposes and strictly not in prodution.

Now, let's go to the first part of our ["Erlang RTS" project](http://vas.io/blog/2012/05/13/erlang-real-time-server-part-1-diameter-server/)
