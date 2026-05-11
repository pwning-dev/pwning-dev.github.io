---
title: "Abusing AI support chatbots to drain wallets"
author: sherlock
date: 2026-05-10 00:00:00 -0700
layout: post
categories: [ai, security]
tags: [ai, chatbot, security]
excerpt: "How unauthenticated chat widgets can drain LLM tokens or subscription credits"
toc: true
---

Ever noticed an LLM support chatbot sitting on a website, facing the Internet, always online, ready to help?
Ask it a question, and it queries an LLM in the background, each response eating up a fraction of a cent from the site owner's wallet.
Now imagine asking that question a thousand times.
The chatbot will most probably not stop you.
But still someone is silently paying for every word they say.
That someone is most likely the owner of the website that integrated the chatbot, while the person typing could be an attacker burning through their budget.

Yigitcan Kaya's IEEE S&P 2026 paper *[When AI Meets the Web: Prompt Injection Risks in Third-Party AI Chatbot Plugins](https://arxiv.org/abs/2511.05797)* got me looking into LLM chatbots.
Large Language Model (LLM) chatbots are web-facing conversational interfaces that route user messages to large language models to generate responses.
With LLMs being knowledge-retrieval systems on steroids, the use of AI-powered chatbots as customer support agents is on the rise.
In fact, a [study](https://www.tidio.com/blog/chatbot-statistics) shows that 82% of customers would first attempt to use an online chatbot over a human agent.
By 2025, a staggering 95% of customer interactions were [predicted](https://masterofcode.com/blog/ai-in-customer-service-statistics) to be handled by AI.
With many businesses adopting AI chatbots for handling customers, tech giants like IBM have tapped into this market with the launch of products like [Watsonx Orchestrate](https://www.ibm.com/products/watsonx-orchestrate/ai-agent-for-customer-service) to serve enterprise customers.

In this research, I tested several popular WordPress AI chatbot plugins and found the same weakness in every one of them: no rate limits, no authentication requirements to initiate a conversation, and no caps on conversation length.
An attacker can drain a site owner's token budget by simply having a long conversation, or automating hundreds of them.
This post breaks down the attack, the vendor responses, and what can be done about it.
Though this research specifically focuses on chatbots, the same findings and guiding principles may also generalize to other web-facing systems that expose AI capabilities to the masses without any guardrails in place. 

## Anatomy of LLM support bots

LLM chatbots are reincarnations of traditional website chatbots.
They typically sit on the bottom right corner of a webpage, either present website visitors with a set of templated questions, or let them drive free-form conversations.
Before AI, human agents handled those interactions, either synchronously or asynchronously.
In contrast, AI-powered chatbots are always online, relentless responders that tap into their backend knowledge base loaded with company-specific and product-specific details.
Many chatbots implement a Retrieval-Augmented Generation (RAG) system; they index site content, documents, ingest pre-configured frequently asked questions (FAQ) and their answers, and fetch relevant snippets from all these sources to include in their responses.

AI chatbots interface with backend LLMs in primarily two ways.
In a Bring-Your-Own-Key (BYOK) configuration, the website owner supplies their own LLM API keys, separately purchased from a third-party cloud-hosted AI vendor like OpenAI or Anthropic.
In Managed-AI-Backend (MAB) deployments, the LLM chatbot service provider hosts the model.
Therefore, the AI cost is baked into the chatbot SaaS (Software as a Service) pricing.
Such chatbots offer subscription tiers in terms of conversation/message or token quotas.

Depending on the web application the chatbots are being integrated into, the chat functionalities may be made available to guest visitors, authenticated users, or both.

## Leaking tokens from your wallet

I noticed that in most LLM chatbot deployments: **1)** all messages in a chat interaction hit the AI backend, and **2)** unauthenticated users, for example, all website visitors are allowed to interact with the chatbots freely, perhaps to improve the outreach of the business by scaling out the support tasks and hoping to offload human support representatives as much as possible.

The unfortunate fallout is that AI chatbots now essentially expose LLM endpoints accessible to the entire Internet, by design.
Unless a guardrail is put in place, an unauthenticated attacker would be able to engage in an arbitrarily long conversation with the chatbot, and slowly yet steadily burn infinite tokens by abusing the chatbot as a gateway to the backend LLM.
The design of the LLM support bots combined with the business decision of keeping the AI features available to the public opens up a giant loophole.
Website owners that deploy such bots are paying for the AI cost, directly (BYOK) or indirectly (MAB).
Therefore, if an AI chatbot does not limit the volume of messages flowing to the LLM, an attacker could leverage that loophole to mount a resource exhaustion attack on the website integrating that chatbot.
Specifically, by repeatedly engaging in AI-powered conversations, an attacker would be able to drain LLM tokens or exhaust the subscription quota.

## What I found when I started chatting

I decided to evaluate a few popular WordPress LLM chat support plugins to validate my hypothesis.
WordPress is a content management system with 60% [market share](https://w3techs.com/technologies/details/cm-wordpress).
Since numerous security analyses (including Kaya's) have previously been conducted on WordPress plugins, it seemed to be the right choice for this research.
First, I searched for the keyword *"Chatbot"* in the WordPress plugin marketplace.
The search returned a ton of plugins.
I tried to filter them by the use of AI, but as you may guess, almost all the top ones have already integrated AI in their workflow.
In the absence of any *"sort by popularity/stars"* filter in the search results, I randomly sampled a few plugins that had 4.5+ stars and 100+ reviews.

I wanted to answer two questions:
1. Can an attacker drain funds through a long AI-powered conversation?
2. If a rate limit exists, can it be bypassed?

Due to ethical concerns, I didn't target any live deployments of these chatbots.
I set up my own testbed by installing the chatbots on a local WordPress instance and completing the necessary configuration to enable AI features.
If the chatbot required its own LLM key, I used the OpenRouter unified LLM API platform, and I trained the AI on any required data sources so it could respond to user requests.
I then visited the website from an Incognito browser and interacted with the chatbot as a normal visitor, asking questions designed to engage the AI agent.
I verified from the chatbot's billing or OpenRouter usage dashboard that the interactions were consuming subscription quota (MAB) or tokens (BYOK).

I noticed two different billing patterns for MAB-style chatbots: one charges per message, while the other bills per conversation or message thread.
Sometimes the billing is async, so you may have to wait for some time to account for the delay before the charges get reflected in your account.
For the first type of billing, I could just continue the conversation to rack up usage.
The second kind of chatbot relies on a cookie to track a chat session.
In fact, one chatbot I tested (C4) allowed configuring a limit on the number of AI conversations that any role, including unauthenticated visitors, could initiate.
Unfortunately, that limit itself didn't do much.
All I had to do was to close the Incognito session, which clears the session-tracking cookies, and repeat the process multiple times.
For the chatbots I tested, I could just ask the same set of questions over and over without triggering any anomaly detection.

By repeatedly initiating AI-driven conversations in short intervals and (optionally) clearing cookies, I confirmed that not only is this attack viable, but also weak rate limits can be trivially bypassed.
What's worse is that an attacker can further leverage automations to multiply the effect.

## Financial impact of an attack

Estimating the true cost of such an attack in advance is nearly impossible due to the many variables involved, including the size of user prompts (which affects input token usage), the length of chatbot responses (which affects output token usage), message frequency, and the backend model powering the chatbot.
However, to provide some perspective, the following is a rough back-of-the-envelope estimate based on hypothetical assumptions and pricing figures.

**BYOK pricing**

OpenAI GPT-4o-mini is a model likely used for customer support chatbots. As of May 10, 2026, gpt-4o-mini-2024-07-18 costs:
- Input: $0.30/1M tokens
- Output: $1.20/1M tokens

Let me assume:
- 200 input tokens per message
- 200 output tokens per message
- 400 total tokens per message

Cost per message:
- Input: 200 * $0.30 / 1,000,000 = $0.00006
- Output: 200 * $1.20 / 1,000,000 = $0.00024
- Total: $0.00030 per message

At 10 messages/minute:
- Per minute: $0.0030
- Per hour: $0.18
- Per day (24h): $4.32
- Per week (7 days): $30.24
- Per month (30 days): $129.60

A small business on a $50/mo OpenAI budget would burn through it in ~12 days, not considering any real traffic from legitimate visitors.

**MAB pricing**

With a typical MAB plan that offers 250 conversations/month for $50, a single automated session could exhaust the entire quota in a day.

## Responsible disclosure

Vendors were contacted through coordinated disclosure; responses varied.
Some acknowledged and planned mitigations; others disputed impact.
The table below summarizes their response, with vendor names anonymized.

| Chatbot | Vendor response |
|---|---|
| C1 | Rejected the report, stating that the behavior is by design. |
| C2 | Accepted the report. Promised to implement a rate limit. No further updates were provided afterward. |
| C3 | Rejected the report due to CVSS base score being below 6.5, denial of service without demonstrated high availability impact. |
| C4 | Accepted the report. They have a role-based rate limit implemented in the plugin. They are thinking of hardening it, though they did not commit to a firm timeline. |
| C5 | Didn't receive any response after multiple follow-ups. |

I did not reach out to additional vendors because denial-of-service (DoS) issues often fall into a gray area where it can be difficult to distinguish between legitimate functionality and abusive behavior.
On top of that, many of these chatbots are developed by solo developers or small software firms that may lack mature processes for handling security reports and incidents, or may not consistently operate with a security-first mindset.
Finding ways to reach out to chatbot developers, often over multiple channels, and debating if it's an oversight or an intended feature is time-consuming with little practical benefit.
Besides, this appears to be a broader design issue that likely impacts a large portion of the chatbot plugin ecosystem.
Instead of continuing to test more chatbots and contacting vendors individually, I believe publishing this blog post would be a more effective way to raise awareness among a wider audience and encourage willing vendors to address this abuse vector.

## Token exhaustion attack on AI systems

With the advent of AI systems, OWASP has expanded their *Top 10 attacks* beyond traditional web applications to LLMs.
The [OWASP Top 10 for Large Language Model (LLM) Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications) is a community-driven initiative identifying the most critical security risks when building and deploying LLM systems.
The attack I just described is recognized in the LLM security world as [Unbounded Consumption](https://genai.owasp.org/llmrisk/llm102025-unbounded-consumption).

> *Unbounded Consumption occurs when a Large Language Model (LLM) application allows users to conduct excessive and uncontrolled inferences, leading to risks such as denial of service (DoS), economic losses, model theft, and service degradation. The high computational demands of LLMs, especially in cloud environments, make them vulnerable to resource exploitation and unauthorized usage.*

One common example of this class of vulnerability is **Denial of Wallet (DoW)**, which refers to attackers exploiting legitimate application flows to drive excessive LLM consumption and financial harm.

> *By initiating a high volume of operations, attackers exploit the cost-per-use model of cloud-based AI services, leading to unsustainable financial burdens on the provider and risking financial ruin.*

The demonstrated attack fits into this category.

## Ways to prevent chatbot abuse

Fully mitigating this attack path is not trivial, especially without negatively impacting chatbot usability.
The preventive measures outlined below can help raise the bar by introducing friction along the attack path, although none of them provides a complete remediation on its own.

**Captcha:**
The simplest and probably the most effective solution to prevent machine-scale abuse.
However, this solution does not work for chatbots that charge per message sent, not per conversation.

**Email verification:**
Making users verify their emails at the start of a conversation ties the chat session to that email.
Like captcha, this solution, too, does not work well for chatbots that charge per message sent, not per conversation.
In addition, chatbots choosing this solution would have to be very careful about whitelisting allowed domains, or else it can trivially be bypassed by minting infinite emails through disposable/personal domains.

**Role-based rate limit:**
Gate AI features behind authentication, where feasible.
Further, if the chatbot has the notion of *roles*, different AI quotas can be allocated to different roles as well.
When the anomaly detection system detects an abuse, the associated user's account can be acted on.

**IP-based rate limit:**
This is weaker than the other options as it can easily be subverted by rotating IPs.

**Real-time anomaly detection:**
Chatbots may monitor token usage and request volumes in real-time, sending automated alerts to website owners or terminating chat sessions for suspicious consumption patterns.

**Warning to the website owners:**
If nothing else, the chatbots should call out the possibility of token abuse at the time of configuring AI features, so that website owners can make an informed decision whether to enable the features or not.
None of the chatbots I looked at displayed any warning at the time of deployment.

## Closing thoughts

Given the pace at which AI is being integrated into production systems, we must carefully vet the newly emerged attack surfaces where traditional systems meet AI.
At the time of this writing, AI cost remains heavily subsidized.
As those costs rise in the future, attacks like this could have a significantly greater impact.
Whether exposing LLM endpoints through chatbot proxies is a sustainable design choice should be evaluated through tomorrow's lens, not today's.
Ultimately, whether this behavior constitutes a bug or a feature is for chatbot vendors to decide.
