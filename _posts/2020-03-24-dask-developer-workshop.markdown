---
layout: single
title:  "Tales from D.C.: Dask Developer Workshop 2020"
date:   2020-03-24 20:00:00 +0100
tags: technology python data-engineering dask distributed
header:
  overlay_image: assets/images/tech_gear_banner.jpg
  overlay_filter: 0.2
  show_overlay_excerpt: false
author: Lucas Rademaker
author_profile: true
---
# Tales from D.C.: Dask Developer Workshop 2020

### Setting
Back in January this year, [Dask](https://dask.org/) community members received an invite for a workshop in Washington DC with the goal of "connecting advanced users with developers".

The schedule of the workshop alone was quite satisfying to me: a number of talks, involving users and developers of Dask and related projects (e.g. NumPy, xarray, Arrow), combined with working sessions, wherein one had the opportunity to discuss or code with other attendees.

### The workshop
My colleague Florian Jetter was the first to present. He outlined our use case of executing Dask pipelines coupled with tight SLA's (Service Level Agreements), which obliges us to make sure that our dask pipelines finish succesuflly each run, as ussually we do not have time to re-schedule. He also talked about some of the issues we had faced in the past and others which we are facing in the present (mostly related to stability).

It was nice to see that we were not the only ones facing these kind of issues. In fact, almost all major issues mentioned at the workshop were shared among multiple attendees. Some of these were:
- Stability: in particular network stability, lack of replication of results, downscaling issues.
- Performance: lack of memory backpressure (OOME's are a thing), other issues involving suboptimal graph optimization  (e.g. task fusing).
- Suboptimal work stealing.

Other issues mentioned involved indexing/partitioning, the lack of a mutli-index and "non-functional" items such as documentation and examples.


The atmosphere of the workshop was great, people were eager to share their experiences with Dask and the ecosystem surrounding it. This provided for some interesting discussions.

### Final remarks
All in all, my colleagues and I found this workshop very valuable. We found this value in being able to share our issues and discuss potential ways of improvement. Additionally, listening to other Dask users and developers gave us a wider perspective on the current status of Dask, and the ecosystem around it, and what we could expect in the future; this allows us to focus on what's most efficient for us now.


Personally, I cannot say it enough times: I loved the working sessions. I learned a lot about Dask and distributed internals and made my first contribution to distributed when I sat down with [John Lee](https://github.com/leej3/) and we made it possible to benchmark work stealing for tasks with restrictions.

Apart from increasing my abilities to contribute upstream, this newly acquired knowledge about Dask has enabled me to aid my colleagues performing data science more effectively when they run into issues with these tools.

Although I secretly hoped that with all these smart people in one room and all the time allocated to working sessions, _all_ major Dask issues would be resolved, I believe a good number of PR's were made during these days, and a lot of valuable knowledge shared which will provide its fruits in the mid-term.

I'm already looking forward to the Dask developer workshop 2021 ;).