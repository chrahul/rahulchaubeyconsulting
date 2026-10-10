---
title: "The AI Assistant That Gives Three People Three Different Answers"
description: "An enterprise architect asked me a hard question about permission-aware RAG. Here is the answer, running."
pubDate: 2026-10-10
tags:
  - Enterprise Architecture
  - AI Security
  - RAG
  - Software Architecture
  - LLM
draft: false
---
Here is a question worth asking before your company approves an AI assistant over internal documents.

A new engineer joins. Three weeks later they ask the assistant a routine question about a past outage. The assistant searches every document the company has indexed, finds the four most relevant passages, and writes a clear, well-structured answer.

One of those four passages came from a restricted incident report.

Who is accountable for that?

Not the engineer. They asked a normal question. Not the model. It has no concept of confidential and no way to acquire one. Not the search, which returned the most relevant results, which is exactly its job.

The failure is architectural, and it happens before anyone writes a prompt.

This article is about where that failure lives and how to remove it. Everything in it was built and measured, and every number comes from a live run on AWS. The code is public.

---

## Why this is not the usual AI security conversation

Most enterprise AI risk conversations are about the model. Will it hallucinate. Will it be jailbroken. Will it say something embarrassing.

Those are real, and they are not this.

This is a plumbing problem, and plumbing problems are the ones that kill pilots at the security review. The model never misbehaves in this scenario. It is handed confidential text and it does the correct thing with it, which is to summarise it accurately for whoever asked.

Everything downstream of retrieval is too late.

You cannot instruct the model to keep a secret it has already been shown. By the time it reads an instruction saying do not reveal confidential information, the confidential text is in the same prompt, a few lines above. You cannot filter the answer afterwards either, because nothing sits between the model and the user that knows what was supposed to be secret.

There is no second line of defence. That is the whole reason this is worth your attention rather than your security team's attention alone.

---

## The structural reason it happens

Every access control system you run today works the same way. Someone asks for a specific thing. Open document 4471. The system checks their badge. They get it, or they do not.

One request. One resource. One decision. Obvious where the check goes.

AI search has no such moment.

The user never names a document. They ask a question, and the system decides for itself which documents are relevant. It may consider hundreds to choose four.

So there is no request to check against a badge. The moment where permission is normally enforced does not exist.

> In a normal system, access control decides what you are allowed to open. In an AI system, it has to decide what the search is allowed to look at. Get that wrong and there is nothing downstream to catch it.

---

## How bad is it really? I measured it.

A reasonable executive response at this point is that this sounds theoretical. It is not, and the measurement is easy.

I built a small corpus for this. Six documents, deliberately mixed. Two public, one internal, two confidential, one restricted. In total they produce seventeen searchable passages.

Then I asked an ordinary question with no filter applied, the way most demos work.

![Search results showing two of four top chunks are confidential](/images/insights/P1-the-danger-measured.png)

Two of the four most relevant results were confidential. One of them was a customer architecture review belonging to a single named account.

Nothing was broken. The search ranked by relevance and returned the best matches. That is what relevance means.

But relevance is not permission, and a system that conflates the two will hand a contractor a customer's architecture review while answering a general question about disaster recovery.

---

## The two obvious fixes, and why both are wrong

### Fix one: store the permissions alongside the documents

Tag each document with who can see it. Filter on that tag when searching.

This works on the first day and degrades every day after.

Tag an incident report with `department = security`. Six months later the company reorganises. The Security team becomes Cyber Defense. Three people leave, two join, one moves business unit.

The document has not changed. But everything your index believes about who may read it is a photograph of a team that no longer exists. The people who left still retrieve it.

That is not a stale cache. It is an authorization failure, and it is silent.

So you fix the drift by syncing permissions continuously from the source systems. And now you are running a permissions system. With its own reconciliation logic, its own edge cases, its own bugs and its own gradual divergence from the systems that actually own those decisions.

Your company already has an identity and access management system. You have just built a second, slightly wrong one and volunteered to keep it in sync forever.

This is exactly the objection that Xin Xing, an Enterprise Data Architect, put to me in public when I first described this design. His question was:

> How do you stop your vector database from becoming a second, gradually wrong IAM system?

I did not have an answer at the time. Most of what follows exists because of that question.

### Fix two: do not store permissions at all

Ask the source systems on every request. Query the identity provider for current group membership. Then the document management system. Then the entitlement service.

Correct in principle. Unusable in practice.

Your query latency is now the sum of several systems' response times. Your availability is the product of their uptime. Four systems at 99.9 percent gives you 99.6 percent before your own code has done anything.

More importantly, you have coupled a platform intended to serve thousands of people to systems you do not own, cannot tune, and will not be consulted about before they are taken down for maintenance.

Two extremes. Both wrong. Which is usually the sign that the question was framed badly.

---

## The principle underneath

I stopped trying to design the rule and went looking for the principle the rule should follow from.

> The search index is exactly that. An index. It is not the source of truth for identity, for authorization, or for document governance.

Everything else is a consequence of that sentence.

**The index stores what a document is, never who can read it.** Classification. Which team owns it. Which customer it belongs to. Which region it came from. A list of people goes stale the day someone changes teams. A classification label is a property of the document itself and rarely changes.

**Teams and customers are stored as stable identifiers, never as display names.** When Security is renamed to Cyber Defense, nothing in the index changes. It still points at the same team identifier, and the identity provider reports who belongs to that team today.

**A person's attributes come from their login token and are never written down by us.** Clearance, teams, customer entitlements. Read at the moment of the question, used, discarded.

![What the index stores about a document versus what a token carries](/images/insights/P3-the-asymmetry.png)

That last point is the one that makes the whole thing work, and it took me a while to see why.

The person side cannot drift. The identity provider re-evaluates those claims every time it issues a token. Someone who leaves a team carries different claims the next time they log in, with no action required by my platform at all.

Only the document side can go stale. That is a much smaller problem than the one I started with.

---

## What I actually built

![Request flow from question to answer across the platform](/images/insights/P2-request-flow.png)

Two flows, and they run at different times.

**Loading documents runs once, whenever a document changes.** The documents sit in object storage alongside a small manifest file that declares what each document is. A job reads the manifest, validates it, splits each document into passages, labels every passage with that document's attributes, converts it into a form that can be searched by meaning, and writes it to the index.

One detail worth noting for anyone who has been through a data governance programme. A document present in storage but absent from the manifest is skipped. It is not ingested with a default. An unclassified document quietly defaulting to public is precisely how leaks happen, so the system fails closed.

The manifest is a reviewed file in version control. That means every classification decision has an author, a date and a diff. When an auditor asks who decided this document was internal and when, that is one command rather than a meeting.

**Answering a question runs on every request.** Five steps, and the order of two of them is the entire point.

The service checks the token is genuine. Signature, expiry, who issued it. Nothing is looked up anywhere, because the token is the source.

It turns the claims in that token into a search filter.

It converts the question into a form the index can compare.

Then it makes one call to the index, passing the question and the filter together.

And it sends only the passages that survived to the model.

The filter is an argument to the search, not a step applied to the search's output. That single difference is what this entire build is about.

---

## Three people, one question, three answers

Here is what that produces. Three users, same question, same system.

**Sam** is a contractor. Public access only, no team, no customer accounts.

**Arjun** is a platform engineer with confidential clearance, on two teams, and assigned to the Apollo customer account.

**Priya** is a security engineer with restricted clearance. The highest in the organisation. Not assigned to any customer account.

The question is the same for all three: what is our approach to disaster recovery testing?

| | Clearance | Passages they can reach | What they got |
|---|---|---|---|
| Sam | public | 5 of 17 | Four public sources. The system said the specific approach was not in what it had. |
| Arjun | confidential | 11 of 17 | The customer architecture review, at the second and third positions |
| Priya | restricted | 14 of 17 | No customer review. Fourth source was an internal platform runbook. |

Read the last row again.

Priya holds the highest clearance in the organisation and does not receive the customer document. Arjun, with lower clearance, does.

Clearance decides how high you can reach. Team and customer assignment decide which documents at that level are actually yours. Those are two different questions and most access models collapse them into one.

Two more things in that table matter more than they look.

Sam received four results, not two. His answer was not degraded because of documents belonging to other people. More on why that matters in a moment.

And when the answer was not in what each person could see, the system said so rather than inventing something. Arjun got a specific recommendation from the customer review. Sam and Priya got an honest statement that the specific approach was not in the material available to them.

---

## The version that passes code review and is still wrong

This is the part I would want a CTO to understand, because it is the version most teams build first and it survives review.

The intuitive implementation searches everything, takes the top results, and then removes the ones the user may not see.

Every permission decision it makes is correct. No confidential text reaches the user. If you read the code you will find nothing wrong with it.

I ran both versions against the same question for Sam, the contractor.

![Filtering before the search compared with filtering after it](/images/insights/P4-prefilter-vs-postfilter.png)

The filtered-first version searched the five passages he is permitted to see and returned four.

The filter-afterwards version searched all seventeen, took the best four, found that two of them were confidential, dropped them, and returned two.

Three things went wrong there, and none of them is a leak of text.

**His answer got worse because of other people's documents.** He asked for four sources and received two. With a narrower corpus he would have received nothing at all, and been told no information exists, while the system holds material he is entirely cleared to read. His experience of the product degrades based on permissions that have nothing to do with him.

**The number itself told him something.** Two results were discarded. That is a fact about documents he cannot see. Ask a question about an incident you are not cleared to know about, and a non-zero discard count confirms the incident exists. Hiding the number does not fully solve it, because the drop in result count and answer quality says the same thing more quietly.

**The confidential text was loaded into the application anyway.** It sat in memory, for an unauthorized user's request, before being dropped. One logging statement, one exception message, one defect in the filter loop, and it is out. With the filter inside the search, it never leaves the database.

The defect is not in a line of code. It is in the order of two steps. That is why it survives code review, and why it is worth a few minutes of a CTO's attention rather than being delegated entirely.

---

## Back to Xin's question

Xin's challenge was that storing permissions in a search index turns it into a second entitlement system that slowly diverges from the real one.

The design claims that only the document side can go stale, and that the person side cannot.

That is a testable claim, so I tested it.

![How an access change and a reclassification take effect differently](/images/insights/P5-two-kinds-of-change.png)

**On the person side**, I issued Priya a new token that added the Apollo customer account, as though the identity provider had granted it. Nothing on the platform changed. No job ran, nothing was re-indexed, no configuration was touched.

Her reachable passages went from 14 to 17 and the customer review appeared in her results on the very next request.

That is the point. Her access is not stored anywhere in my system, so there was nothing to update.

**On the document side**, I reclassified the customer review from confidential to internal by editing the manifest.

Before re-ingestion, Priya still saw 14 passages and no customer review. The source of truth had changed and the index had not. That is the stale window, and it is real.

After re-ingestion, 17 passages, with the document correctly labelled internal.

| What changed | Where it lives | What had to happen | When it took effect |
|---|---|---|---|
| A person joins an account | their login token | nothing on the platform | the next request |
| A document is reclassified | a reviewed manifest file | one re-ingestion | after re-indexing |

So the honest answer to Xin is this. The index does hold something that can lag behind reality. But it holds document labels, not people. The lag is one known, visible, repeatable step driven by a file somebody reviewed, rather than a sync job quietly copying the company's entitlements into a second place where they rot.

Xin then wrote his own piece about the exchange and improved the design further in three ways I had not considered. Those are going into the next decision record rather than being quietly absorbed. The most useful of them: when a live permission check is eventually added for high-consequence documents, it should fail closed per passage rather than per request. A query returning three sources instead of four is degraded. A query returning nothing is an outage.

---

## What is still open

I would rather publish the gaps than present a finished-looking design, partly because the gaps are where the interesting work is.

**What happens when a live permission check times out.** Fail closed and a slow identity provider can take the platform down. Fail open and the control disables itself under load, which is exactly when you need it. The likely answer is fail closed with a short cache, and I want to build it before committing to it in writing.

**How often is often enough.** Some documents change their permissions constantly and matter little. Others never change and matter enormously. How likely the stored copy is to be wrong and how much damage being wrong causes are different questions, and both have to drive the decision. At the moment this is declared as policy per document class. It should eventually be measured.

**Reclassification can fail in a dangerous direction.** Loosening a document's classification shows it late, which is harmless. Tightening one leaves it visible to too many people until the next re-ingestion. The ordering guarantees around that are not yet designed.

And several deliberate simplifications that I would not take to production. The token signing is symmetric, which means the holder of the secret could mint tokens. A real deployment uses the identity provider's private key and a published public key, so the service can verify tokens but never create them. The network layout favours cost over isolation. The infrastructure state lives in one file on one machine.

None of those change the authorization argument. All of them would change before anyone put real documents in it.

---

## What this means if you are approving an AI programme

Four things, and none of them is about which model to use.

**Ask where the permission check happens.** Not whether there is one. Where. If the answer is that results are filtered after retrieval, you have the version that passes code review and still tells people what they cannot see.

**Ask what the AI system stores about people.** If it stores group membership or access lists, someone now owns keeping that in sync with your identity provider forever. That is a long-term operational commitment that rarely appears in a project plan.

**Ask who classified each document, and where that is recorded.** If the answer is a folder structure or an engineer's judgement at upload time, you have no audit trail. A reviewed file in version control gives you an author, a date and a history for every decision.

**Ask what happens when someone leaves a team.** If the answer involves a sync job, ask how long the window is and who watches it. If the answer is that it takes effect on their next login, the design is probably sound.

None of these are AI questions. They are the questions your organisation already asks about every other system that touches sensitive data. The reason pilots stall is that AI projects are often reviewed as technology experiments rather than as systems handling data people are not all cleared to read.

---

## The honest summary

What I built is a reference implementation, not a product. It runs on a disposable cluster that I create for a session and destroy afterwards. The documents are synthetic. Several components are deliberately simplified and I have listed them.

What it demonstrates is not simplified. Three people ask one question and receive three different answers, grounded only in what each is permitted to read, with the person holding the highest clearance correctly receiving less than a colleague with lower clearance. The retrieval and authorization code that produces that result is unchanged from the version I first wrote on a laptop.

And the question that prompted all of it, from someone who disagreed with my first design in public, now has an answer you can run rather than an argument you have to accept.

That is the part I would keep.

---

*Thank you to Xin Xing, whose question changed this design and whose follow-up improved it again. The repository, the decision record and a step-by-step tutorial with every command and its real output are at github.com/chrahul/production-rag-eks.*
