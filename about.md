---
title: About me
permalink: /about/
layout: page
excerpt: "About me"
comments: false
---
{% assign dateStart = "2001-09-26" | date: '%s' %}
{% assign nowTimestamp = 'now' | date: '%s' %}

{% assign diffSeconds = nowTimestamp | minus: dateStart %}
{% assign diffDays = diffSeconds | divided_by: 3600 | divided_by: 24 | divided_by: 365 %}

My name is Pedro Ferreira and I'm {{ diffDays | round: 0 }} years old. For two years I attended with approval the Computational Engineering course of University of Aveiro, having changed in 2021 to the BSc in Computer and Informatics Engineering.

During this time, I learnt about algorithms and data structures, how to program in Python, Matlab and Java, analyze and manipulate data to model physical systems, use Git repositories for collaborative work and create and manage Docker containers.

In my free time I enjoy playing guitar, listening to music and creating new things (informatics-related projects). 