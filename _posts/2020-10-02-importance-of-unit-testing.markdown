---
layout: post
title:  "Importance of Automated Testing"
date:   2020-10-02 16:14:42 +0530
categories: programming coding unit-test functional testing
---

Automated testing (which includes unit and functional tests) refer to the framework
of writing and running tests against the codebase. There is a lot of debate out there
on how useful or rewarding these tests really are. Questions like "Are unit tests worth
it?" or "Should I be writing unit tests?" keep on popping up. I am a strong advocate of
automated tests. Any part of code which can be tested automatically should have a test
case written for it. In this post I have tried to write down the benefits of having
well-written test cases. Some of these points are very apparent and logical but some
rewards of automated testing are only reaped over time.

### You are already doing it

Have you ever, after writing a functionality, tested it by launching an interpreter
shell in your terminal? Or perhaps tied the method call to a button click and see
what it prints or alerts? If yes you are already doing unit testing. If you are developing
APIs you are probably using Postman or CURL to hit your dev endpoint with expected
requests and verifying the response.

Unit testing is taking that behaviour and saving it so that the tests can be run on demand. If 
you write a "proper" test case rather than manually testing, it takes the same amount of 
time, or maybe less as you get experienced, and you have it available to repeat again and again.

### It saves time and money

One of the reasons people shy away from writing unit/functional tests is that it takes time.
Have you ever spent time debugging an application in production? Ask yourself these questions:

- How much time did it take to narrow down the root cause?
- How much time did you take to fix it? Once fixed how did you verify that it is fixed?
- How much revenue did it cost to the business?
- Did it result in loss of trust or reputation with your customers?

Well written unit tests will help you minimise downtimes due to logic bugs. They provide
you with a framework to persist the knowledge of a bug even after it is long forgotten.

### An opportunity to spot useless code

Ever written a piece of code or an if/else which you beleived was necessary but would never
be run in production. Automated tests with coverage reports will spot such lines and help
you identify possibly useless routines or unused classes early in the development cycle. It
provides you with an opportunity to identify code whose utility might not be
apparent at first sight.

### Ship with confidence

This is a no-brainer. If you have "well-written" test cases and if these cases are passing
on a commit you can be sure that the functionality works. You need not fear if this API will
work when deployed. 

### Document how the code is intended to be used

Unit tests also serve as examples of how the methods can be called and how the classes are
intended to be initialized. This is very helpful for someone who is new to your codebase.
A well organized test suite is a playground for someone who intends to understand how things
work.

### Lose the fear of making changes

We all have been to a place when we had to change the behaviour of a function. Normally we
need to go down the rabbit hole in search of all the origin nodes from where the target method
can be reached. Its tedious and prone to errors. What if you missed one entry-point or API call?
Automated tests squashes this fear. You can freely go ahead and change the behaviour and run
your tests. If they pass you can be confident that nothing is broken.

### No resurfacing bugs

Resurfacing bugs are a common occurrence. A bug is reported, someone fixes it and the release
is deployed. Everyone is happy. Yay! But one month down the line the bug is introduced again.
The developer generally goes through the phases of denial, confusion, anger and acceptance. What
if this can be avoided?

Once you spot a bug after deployment you should write a test case for it. This test case will be
run against all the future builds of the product and you would have made sure that a bug once
fixed does not cost you time to fix again.

### Upgrade dependencies with confidence

Once your software has reached a certain complexity and is supporting some serious business
dependency upgrades are a reality. New vulnerabilities are detected which can seriously impact
the security posture of the business. In some cases new features or functionalities are added
which are must for your use-case or increase efficiency multi-folds. Consider these scenarios:

- I designed my app to work with MySQL 5.7. I need to upgrade to MySQL 8 to use window functions.
Will my app work with MySQL 8?
- A serious vulnerability has been detected in my libcrypto-2.0.9. I need libcrypto-3.0.0 to
mitigate it. Will my app break with 3.0.0?

These questions can be answered by a single run of test case against the new database endpoint
or new dependency. These scenarios are inevitable with time. In many products these
questions are never asked and avoided because the answers are too difficult to arrive at and
the product and the developers suffer.

### Refactor as much as you need

This is in continuation to "Lose the fear of making changes". The impact of the ability to
refactor the code "mercilessly" is difficult to put out in numbers unless you try it out. Need
to move functions across classes, change inheritance pattern or break down a mammoth file/class
into multiple modules. These tasks are daunting not because of lack of skill or ability but due
to stress and fear of making irreversible damage to codebase and losing time. Test cases will
allow you to continuously test the progress of refactoring over smaller refactors.

## Conclusion

Automated tests are industry-best-practices for a reason. If you are starting a new project
you should strongly consider having test cases written. It is very easy to include the testing
workflow when the project is starting than to introduce it to an already mature codebase.
