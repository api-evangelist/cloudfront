---
title: "Reduce Traffic Interruptions with Gateway Load Balancer TCP Reset"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/reduce-traffic-interruptions-with-gateway-load-balancer-tcp-reset/"
date: "2026-09-02"
author: "Donathan Ratcliffe"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/feed/"
---
When a firewall or security appliance behind your Gateway Load Balancer (GWLB) fails, what happens to the TCP connections flowing through it, and how long do those traffic interruptions last? Until today, they could hang while TCP retry mechanisms and exponential backoff ran their course, sometimes for minutes. For mission-critical applications, every second of interruption […]
