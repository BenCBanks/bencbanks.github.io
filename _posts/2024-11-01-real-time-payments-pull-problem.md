---
layout: post
title: Real-Time Payments and the Pull Problem
type: Essay
description: RTP and FedNow are live, but adoption has been uneven. The interesting question is not whether real-time payments will win — it is how long it takes for pull payments to catch up with push.
tags: [payments, fintech]
hide_image: true
---

RTP and FedNow are live. The rails exist. But adoption has been uneven, and the reason is straightforward: real-time push payments are easy, and real-time pull payments are hard.

Push is simple. A sender initiates the transaction. The money moves. Done. This is how Zelle works, how most payroll use cases work, how insurance disbursements work. The payer controls the timing, and that makes the flow easy to reason about.

Pull is the problem. Pull payments — where the payee initiates the debit — require a layer of authorization, trust, and dispute resolution that the current real-time rails weren't built to handle gracefully. ACH has decades of infrastructure around returns, reversals, and NACHA rules. Real-time rails have none of that yet, and building it takes time.

This is why the most interesting question in real-time payments is not whether the rails will win. They will. It is whether the ecosystem — banks, fintechs, payment processors — can build the authorization and risk infrastructure needed to make pull as reliable as push.

Until that is solved, real-time payments will remain a one-way street for most use cases.
