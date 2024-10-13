---
title: "work"
description: "My long-form resume with more details about my past professional and academic experience."
summary: "This page aims to document my past work and academic experiences in higher detail. You can think of it as a long-form resume."
lastmod: 2024-09-02
layout: single
---

# High school
<p class="subtitle">early 2016 to late 2018</p>

I first got in contact with programming during my high school program at [COTUCA](https://cotuca.unicamp.br/cotuca/). There, I mostly played around with web development in PHP, NodeJS and raw HTML5/CSS/JS. I made a few games which have been long forgotten, and tried to get my head around how machine learning worked, though I mostly failed due my lacking knowledge of calculus and linear algebra at the time.

During that time, I worked as a student assistant, and would help other students with their programming assignments.

Despite being formational years in my career as a Software Engineer, I will not elaborate more about the projects I worked on during high school for brevity.


# University
<p class="subtitle">early 2019 to mid 2023</p>

During my 5 years as a Computer Science student at [UNICAMP](https://unicamp.br/), I worked on assignments, some small personal projects and a few larger ones. Apart from that, during most of my education, I also worked as a backend software developer for [Wavy](#wavy) and [Transform Data](#transform-data).


## Autonomous racing car telemetry
<p class="subtitle">early 2020 to mid 2022</p>

[Unicamp E-Racing](https://www.unicamperacing.com/) is my university's Formula Student team for the electric cars division. 

My main body of work at the team was the autonomous racing car's telemetry. In short, our goal was to provide near real-time metrics of the race car.

For that, me and the rest of the telemetry team had to write
- **The agent**: a lightweight Python script that ran on the car's main board (NVidia Jetson Xavier) to collect CAN bus data, perform aggregation, categorize the data and send it in batches via a custom UDP-based network protocol.
- **The ingestion controller**: another Python service that ran on the main base-station computer and would listen for UDP packets. It decompressed the incoming data and added it to a hashtable of ordered datapoints on our local Redis, while also updating an ordered set of batches used for indexing.
- **The read API**: a REST API that would query the local Redis for snapshots of the data given a time window. It also had a websocket interface for subscribing to new datapoint batches belonging to any metric.
- **The visualization plugins**: a custom set of [Grafana](https://grafana.com/) plugins for connecting to our custom datasource API and visualize racing-car specific data such as the current planned trajectory of the autonomous driving software (using [PIXI.js](https://pixijs.com/) and TypeScript).

At the time, we did look into other metric-aggregators like Prometheus and Graphite, but none of them really fit our exact needs. Some were open source but lacked features like querying at very small granularities (sub-millisecond), and and others were expensive proprietary solutions or unmaintained projects.

Fortunately, nowadays there are tools that fulfill those requirements such as [rerun](https://rerun.io/). It was a very fun project where I learned a lot regardless.


## Automated detection of faulty soldering in electronics
<p class="subtitle">early 2023 to mid 2023</p>

During my 1 year exchange to [Umeå University](https://www.umu.se/), as a part of an applied computer vision course, I was a part of a team which investigated the performance of different computer vision techniques when applied to detecting faulty soldering in electronics. 

The main goal was to assist a partner company (one key vendor of electronic components for industrial forestry machines) with the development of a better quality assurance pipeline that needed less human intervention. To reduce the problem complexity, our project was scoped at only one of their most manufactured boards, which contained a limited amount of solders in pre-determined positions.

This project had 3 main phases:
1. **Assembling a diverse dataset**: During this phase, we took thousands of pictures of our target component, and categorized all of them according to different control variables such as camera type (webcam, iPhone, DSLR), camera distance, camera angle, light color, ambient light (controlled, mild, intense).
2. **Investigation of localization algorithms**: During this phase, we investigated the performance of different algorithms with _localizing_ all the interest points in the picture, so that they could be cropped and classified. The best performing algorithm was SIFT, but we also tried ORB, and combinations of simpler transforms and color thresholding.
2. **Investigation of classification algorithms**: During this phase, we developed a simple classifier that would determine how much of the electronic connector was uncovered by the solder by applying a simple color mask and counting pixels. It was surprisingly effective and compute-efficient compared to our other machine-learning base algorithm, which performed better but required more compute. We concluded that the camera type, distance and ambient light color could seriously influence classifier efficiency.

The final report of this investigation was forwarded to the partner company.


# Wavy
<p class="subtitle">early 2019 to late 2020</p>

Wavy (later acquired by [Sinch](https://www.sinch.com/)) was the largest enterprise messaging provider in Latin America, providing automated customer service solutions over SMS, WhatsApp, RCS and other text communication channels.

I started my career there as an intern right after High School, and progressed to Software Engineer while I worked there.

## Chatbot analytics platform

At Wavy, the number of chatbots was growing, and we needed a scalable way of tracking interactions and billing. I was tasked with solving this problem.

The solution I came up with was a simple event ingestion AWS Lambda endpoint that would take in "events" and publish them to an SQS queue. At the other end of the queue, there would be a batch writer which would ingest the data onto a BigQuery `chatbot_events` table, which had `created_at`, `type`, `bot_id`, `converstion_id` and `payload` fields. This allowed the company to turn questions such as "How many items were added to the cart this month in customer X's bot?" into SQL that ran in our analytics database.

I didn't know it at the time, but this simple system ended up being perfected over time and it became the company-wide de-facto way of tracking bot analytics.


## Automating customer service of large food marketplace

One of our key accounts was [iFood](https://www.ifood.com.br/), a large online food marketplace which was growing a lot at the time. With growth, came the challenge of scaling their customer service department to handle thousands of support tickets daily.

To assist them with this project, we developed a chatbot using Google's Dialogflow, which would connect to their Zendesk instance and perform tasks like triaging tickets, automatically asking for documentation upfront before handing off to a human agent, and performing requests on their backend APIs.

The chatbot was quite large (over 500 NLP intents, more than 100 different flows), and it helped them improve their average ticket resolution time from over 30min to 5min, and increased customer satisfaction metrics by over 30%. It was also a significant cost-saver since it greatly improved the efficiency of their customer support agents.


## Ordering food through messaging

The final project I worked at Wavy was also in collaboration with iFood. It consisted in providing a messaging-first experience to ordering food. We developed an in-house chatbot engine that would integrate with NLP providers and iFood's backend APIs to allow customers to turn sentences such as "I would like to order a pizza" into an actual order in their API. 

Our main challenge was structured search, since the same product could be represented very differently by different restaurants, for example:
- Restaurant A would categorize `Large Pizza` and `Small Pizza` as different `products`, and the toppings would be `variants` of such products.
- Restaurant B would categorize `Margheritta Pizza` and `Pepperoni Pizza` as different `products`, and the sizes would be the `variants`.

To solve that, we experimented with a lot of different options, but ended up settling with representing every product/variant as a string concatenation of all its attributes, and then fuzzy matching against it based on the extracted entities.

The project was very experimental, and it ended up being discontinued after a while.


# Transform Data
<p class="subtitle">early 2022 to early 2023</p>

Transform Data (later acquired by [dbt Labs](https://www.getdbt.com/))

## Slack integration

I started my journey at Transform by writing a custom Slack integration that could render charts and send them as embedded images in Slack. This project involved:
- Performing OAuth 2.0 with the Slack API, including token rotation and invalidation.
- Fetching time-series data from our internal metrics APIs.
- Spinning up a headless Chromium instance that could load the chart renderer page, hydrate it with data and take a screenshot using [pyppeteer](https://github.com/pyppeteer/pyppeteer).
- Managing AWS S3 artifacts.
- Sending messages through the Slack API.

Despite those technical challenges, my main challenge for this project was communication. At the time, I was just a contractor and not yet a full-time employee, so I didn't have the same level of access and permissions as a regular employee. As such, the hardest part was coordinating with everyone so that the Slack integration service could talk to the backend monolith and the frontend renderer.

## Boards

At Transform, we were building a Semantic Layer (which I still work on at dbt Labs!), through which users could define and query their business metrics. At the time, we were building our own querying interface which was similar to a BI tool. It allowed users to pick from different chart types, slice by different dimensions and save their queries. One of the components of such interface was named "Boards", and it gave users the ability to create and save their own dashboards with many charts, sections etc.

Me and another backend engineer were responsible for the database and APIs. We chose to model a dashboard as a tree of components, since it could contain sections within sections with many charts. In the Postgres database, each Board's metadata was stored in a `boards` table, and each node contained in a dashboard was stored in a `board_nodes` table, which was indexed by `board_id`. This made it easy to reconstruct the tree from top to bottom using a single SQL query similar to the following

```sql
SELECT 
  id, depth, parent_id, type, title, ...
FROM 
  board_nodes
WHERE 
  board_id = $board_id
ORDER BY 
  depth
```

We contemplated performing data validation to ensure components wouldn't overlap, but that involved a lot of geometric computations that were not trivial for the backend. Instead, the engineering team agreed that the frontend would be responsible for submitting geometrically-coherent data, and it would self-heal in case the data stored in the database was wrong.

The API layer was a simple GraphQL wrapper around those tables, through which the frontend could query for `boards`, `boards.nodes` and mutate a dashboard via mutations such as `boardsCreate`, `boardsDeleteNodes`, `boardsUpdateNodes` etc.

## GraphQL diff tool

At some point, I got the feedback from my manager that one of the expectations for Senior engineers is to have more impact across the organization. Meanwhile, I noticed our frontend team would always complain that the backend team broke their code by making backward-incompatible changes to our GraphQL APIs.

I saw an opportunity of having more impact by writing a simple look-ahead parser that would generate a list of breaking changes made to a GraphQL schema based on the previous schema. I then wrapped it into a simple Github Actions bot that would automatically comment that list on PRs that introduced breaking changes to the API, and request a mandatory approval from the frontend team on that PR. 

Despite being a simple hacky solution that was at most a few hundred lines long, the impact on the "happiness factor" of the frontend team was great. By doing this, I learned that next-level backend engineers like the one I aspire to be maximize their impact by helping others do their job better.


# dbt Labs
<p class="subtitle">early 2024 to present</p>

Transform Data was acquired by dbt Labs, and, after some rearrangements, finishing college and moving to a new country (that's why there's a 1-year gap between Transform Data and dbt Labs), I now work there on their [Semantic Layer](https://www.getdbt.com/product/semantic-layer).

So far, I've contributed to [MetricFlow](https://github.com/dbt-labs/metricflow) and the open source Semantic Layer [Python SDK](https://github.com/dbt-labs/semantic-layer-sdk-python/). The rest of the work I am currently performing at dbt Labs is a little too recent to talk about publicly, but I promise it's exciting stuff!
