---
layout: page
title: Workerholic
subtitle: How we built a Background Job Processor from scratch
use-site-title: true
---

This is the story of how three Software Engineers decided to shed some light on Background Job Processors by building one from scratch.
We started by understanding the use for a Background Job Processor (BJP), then, identifying the most popular ones in the Ruby-ecosystem and, finally, building one from scratch.
In this post we will walk you through the proces of building a BJP from scratch and how we managed to build one, **Workerholic**, that compares to Sidekiq in terms of performance.

But first, let us introduce ouselves:

[![antoine](img/antoine.jpeg){:width="240"}](https://antoineleclercq.github.io)
[![konstantin](img/konstantin.jpeg){:width="240"}](http://minevskiy.com/)
![timmy](img/timmy.png){:width="240"}

<ul class="team-names">
  <li class="team-member-name">
    <a href="https://antoineleclercq.github.io">
      <strong>Antoine Leclercq</strong>
      <br>
      Software Engineer
    </a>
  </li>
  <li class="team-member-name">
    <a href="http://minevskiy.com/">
      <strong>Konstantin Minevskiy</strong>
      <br>
      Software Engineer
    </a>
  </li>
  <li class="team-member-name">
    <a href="">
      <strong>Timmy Lee</strong>
      <br>
      Software Engineer
    </a>
  </li>
</ul>

We are three Software Engineers **looking for our next opportunity**. To learn more about us you can **click us**!

First, here is a quick demo of **Workerholic in action**:

![demo_workerholic](img/demo_workerholic_0.gif){:id="demo"}

{% include_relative toc.md %}
{% include_relative story.md %}
