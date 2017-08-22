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

<ul class="team-names">
  <li class="team-member-name">
    <a href="https://antoineleclercq.github.io">
      <figure>
        <img alt="Antoine Leclercq" src="img/antoine.jpeg">
      </figure>
      <strong>Antoine Leclercq</strong>
      <br>
      Software Engineer
    </a>
  </li>
  <li class="team-member-name">
    <a href="http://minevskiy.com/">
      <figure>
        <img alt="Konstantin Minevskiy" src="img/konstantin.jpeg">
      </figure>
      <strong>Konstantin Minevskiy</strong>
      <br>
      Software Engineer
    </a>
  </li>
  <li class="team-member-name">
    <a href="https://tim-lee92.github.io/">
      <figure>
        <img alt="Timmy Lee" src="img/timmy.png">
      </figure>
      <strong>Timmy Lee</strong>
      <br>
      Software Engineer
    </a>
  </li>
</ul>

**We are looking for our next opportunity**, so don't hesitate to **click us to get in touch**!

First, here is a quick demo of **Workerholic in action**:

![demo_workerholic](img/demo_workerholic_0.gif){:id="demo"}

{% include_relative toc.md %}
{% include_relative story.md %}
