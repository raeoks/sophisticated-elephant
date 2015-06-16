---
layout: post
title: "Entropy &amp; Broken Windows"
date: 2014-10-15 18:03:18 +0530
comments: true
categories: [engineering, programming]
---

While looking from a 30,000 feet view, most of the physical systems no matter
how complex they are; appear to have a great sustainability. Software is no
different, while it's immune to almost all physical laws, but entropy still
applies to it.

> In thermodynamics, entropy is a measure of the number of specific ways in
> which a thermodynamic system may be arranged, commonly understood as a
> measure of disorder.

When this measure of disorder increases in a software system, then it's 
called _software rot_.

Out of many factors that contributes to software rot, one that seems to
be crucial is project's psychology or work culture around it. A project's
psychology completely defines its development pace and milestones. A project
with a negative psychology or poor work culture around it results in a failure;
while another project with positive psychology or good work culture, tends to
survive even in the dire consequences. Yet to overcome the project's psychology
issues one must select the team of developers based on how they'll cope with
the project deadlines. This team of developers is need to be well disciplined
and should share responsibility for their actions other than blaming each other
to meet those project deadlines.

## Don't Live with Broken Windows

Researchers have discovered a trigger that leads to urban decay.

{% blockquote Wilson/Kelling, Atlantic Monthly %}
Once a window in a building is broken and left unrepaired, the building
starts to go downhill rapidly. A car can be left on a street for a week,
but break one of its windows, and it will be gutted in hours.
{% endblockquote %}

Similar things also apply to software projects, where broken windows are
replaced by bugs. When one is under market pressure, then the project
deadlines and estimates are calculated to maximize profit by sacrificing
codebase design. Or, when software project's critical decisions taken in
haste without considering its consequences leaving many broken windows.
Time is also one of the constraints that directly decides the number of
broken windows or bugs that can be fixed before shipping out the product.

## Fixing or Avoiding Broken Windows
Since, Don't Live with Broken Windows happens to be an issue for most of the
software projects. This contributes bad designs, wrong decisions and poor
code quality.

* Bad designs can be avoided by using a better methodology of software
  development like test driven development and behavior driven development.

* Wrong decisions are the result of time shortage, where all the solution
  paths weren't discovered before the final decision was made. Just to
  avoid such a wrong decisions in code, developer can comment out offending
  code or display _Not Implemented_ or substitute with dummy data instead.

* Poor code quality can also be avoided by refactoring mercilessly when code
  smells.
