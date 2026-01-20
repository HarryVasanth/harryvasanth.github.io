---
layout: post
title: "DevOps - Grounding the LLM using a 1986 Aircraft Manual"
date: 2026-01-20 14:59:58 +01
categories: devops documentation
tags: ai automation documentation  
---


## The Pervasive Issue of 'AI Slop'

The integration of Large Language Models (LLMs) into our daily engineering workflows has fundamentally changed how we operate. We use these tools to generate code, draft pull requests, and write documentation. However, this automation has introduced a severe degradation in text quality. The industry has coined a term for this phenomenon: "AI slop".

AI slop refers to the verbose, overly polite, and fluffy language that modern AI models default to. It is characterised by complex vocabulary, lengthy run-on sentences, and an overwhelming reliance on marketing adjectives. If we ask an AI to write a simple API configuration guide, it often returns a manifesto about "seamlessly unlocking robust capabilities". This destroys the readability of technical documentation and drastically increases the cognitive load for the reader.

Instructing the AI to "be concise" or "stop using marketing words" rarely yields consistent results. Negative constraints are weak. The AI treats these instructions as mere suggestions and quickly reverts to its base training. To fix the form of the output, we cannot rely on vague prompts. We need to provide the AI with a rigid, rule-based system.

## The Aerospace Solution: ASD-STE100

The cure for AI slop is not a new prompt engineering trick. The cure is a strict linguistic standard developed in 1986.

ASD-STE100, known as Simplified Technical English (STE), was originally developed for the aerospace industry. When writing aircraft maintenance manuals, linguistic ambiguity can be fatal. STE was designed to remove all ambiguity, strip out unnecessary vocabulary, and enforce a highly mechanical sentence structure.

Because STE is a mechanical and rule-based system, LLMs understand it perfectly. By explicitly instructing the AI to adopt the ASD-STE100 standard, we force it to drop the fluffy marketing copy. The result is crisp, highly readable technical prose.

## Core Principles of Simplified Technical English

To cure AI slop, we must instruct our tools to follow the three core pillars of STE. These pillars cover vocabulary, grammar, and structure. When we apply these rules, the text becomes predictable and efficient.

### 1. Vocabulary Restrictions

LLMs love to use ten words where one will suffice. They also love to use synonyms to make the text sound more "dynamic". In technical writing, synonyms cause confusion. STE enforces a strict dictionary and limits word choice to the absolute basics.

* **One Name, One Thing**: We must use one name for one thing consistently. Do not call an item a "server" in one sentence and a "node" in the next. Pick one noun and stick with it.
* **One Word, One Meaning**: We must give each word one specific meaning. For example, the word "fall" means to move down due to gravity. It does not mean "to decrease".
* **Short, Common Words**: We must instruct the AI to abandon its bloated vocabulary.
* **Use, not utilize**: The word "use" replaces "utilize", "leverage", or "employ".
* **Start, not commence**: The word "start" replaces "begin", "commence", or "initiate".
* **Show, not demonstrate**: The word "show" replaces "demonstrate".
* **Before, not prior**: The word "before" replaces "prior to".
* **No Marketing Adjectives**: We strictly forbid words like seamless, robust, powerful, effortless, world-class, and revolutionary. These words add no technical value.

### 2. Grammatical Discipline

The most prominent feature of AI slop is the passive voice. The passive voice hides the actor and makes sentences unnecessarily long. STE eliminates this and enforces direct action.

* **Active Voice Only**: Sentences must describe who is doing the action. Write "the parser reads the file". Do not write "the file is read by the parser".
* **Action Verbs**: We must eliminate nominalisation. Nominalisation occurs when a verb is turned into a noun. Use a verb for an action. Write "analyse the log". Do not write "perform an analysis of the log".
* **No Stacked Auxiliaries**: We must remove filler phrases. Instead of writing "it is important to note that this may help to improve", simply write "this improves X".
* **No Present Participles**: We avoid using "-ing" main verbs where a simple tense works perfectly well.

### 3. Structural Limits

LLMs tend to generate massive walls of text. These text blocks are intimidating and difficult to scan. STE imposes hard mathematical limits on structure.

* **Sentence Length**: An instructional sentence must not exceed 20 words. A descriptive sentence must not exceed 25 words. This forces the writer to break down complex thoughts.
* **Paragraph Length**: We enforce one topic per paragraph. A paragraph must have a maximum of six sentences.
* **Direct Instructions**: We restrict sentences to one instruction per sentence. Do not chain multiple actions together with conjunctions.
* **Punctuation Limits**: Semicolons are strictly forbidden. The AI must write two separate sentences instead. Contractions are also forbidden.
* **List Formatting**: For steps, the AI must use a numbered vertical list. It must use the imperative form and place one action per item. Conditions must always be placed before the command.

## Modes of Operation

When building system prompts for our CI/CD pipelines or local development agents, we apply STE in two distinct modes. The mode depends on the context of the writing task.

### 1. Strict Mode

This mode is used for procedures, runbooks, safety text, and error messages. The AI must apply every single rule and strictly adhere to both length caps. There is no room for creative interpretation. The goal is absolute clarity and zero ambiguity.

### 2. STE-Flavoured Mode

This mode is used for general prose like README files, PR descriptions, and standard architectural documentation. We apply the sentence caps, paragraph limits, active-voice constraints, and verb discipline. However, we relax the strict dictionary lockdown. This ensures the text retains enough range to read naturally while remaining highly structured.

## The System Prompt: 'STE-Writing'

We can encapsulate all these rules into a single, highly effective system prompt. You can save this configuration into your local agent or inject it into your automated documentation pipelines. This skill fixes the form of the text. It applies to documentation, READMEs, pull-request text, error messages, release notes, and comments. It does not apply to code, identifiers, or command syntax.

```text
# Role
Write prose in ASD-STE100 Simplified Technical English. This applies to documentation, READMEs, pull-request text, error messages, release notes, and comments. It does not apply to code, identifiers, or command syntax. It is not for marketing copy, essays, or anything that needs a voice. STE strips voice on purpose.

## Rules
WORDS
- Use one name for one thing. Do not call the same item by two different names.
- Use the short common word: start (not begin/commence/initiate), use (not utilize/leverage), help (not facilitate), make sure (not ensure), before (not prior to), after (not subsequent to), about (not regarding/concerning), get (not obtain/acquire), show (not demonstrate), also (not additionally/furthermore/moreover).
- Give each word one meaning. "fall" means to move down, not to decrease.
- No marketing adjectives: seamless, robust, powerful, cutting-edge, effortless, world-class, next-generation, revolutionary.
- British English spelling (e.g., standardise, optimise, colour).

VERBS
- Active voice. "the parser reads the file", not "the file is read by the parser".
- Use a verb for an action. "analyse the log", not "perform an analysis of the log".
- No stacked auxiliaries. Not "it is important to note that this may help to improve". Write "this improves X".
- No "-ing" main verb where a simple tense works.

SENTENCES
- One instruction per sentence. Max 20 words (instruction), max 25 (descriptive).
- No contractions. Use articles: a, an, the, this, these.

PUNCTUATION
- No semicolons. Write two sentences.
- No em dashes or en dashes used as punctuation.

STRUCTURE
- One topic per paragraph, max six sentences. 
- For steps, use a numbered vertical list, one action per item, imperative form. 
- Put a condition before its command. 
- Write only the requested text. No preamble, no summary, no closing remarks.

## Modes
- strict: procedures, runbooks, safety text, error messages. Apply every rule and both length caps.
- STE-flavoured: general prose (READMEs, PR descriptions, docs). Apply the sentence, paragraph, active-voice, and verb discipline. Relax the dictionary lockdown so the text keeps enough range to read naturally.

## Self-Linting Checklist (Run silently before outputting)
1. Is any sentence over 20 words? Split it.
2. Are there any semicolons or dash punctuation marks? Remove them.
3. Are there any contractions? Expand them.
4. Is there passive voice with a known actor? Make it active.
5. Is there an "-ing" main verb, nominalisation, or phrasal verb? Replace with a plain verb.
6. Is the same concept named two different ways? Pick one name.

```

## Real-World Application: Before and After

To understand the profound impact of this prompt, we can look at some standard AI outputs compared to their STE-corrected versions. These are entirely new examples designed to showcase typical infrastructure and development tasks.

Notice how the baseline examples rely on lengthy joined sentences and empty buzzwords. The STE versions strip this away to reveal the actual technical payload.

### Example 1: The Pull Request Description

Developers frequently use LLMs to summarise their code changes. The standard AI output for a pull request is often a dense wall of sycophantic praise for the developer's own code.

**The Baseline AI Output:**

> This PR seamlessly refactors our legacy database migration scripts to leverage a more robust and cutting-edge asynchronous processing model. Furthermore, it is important to note that the utilisation of this new paradigm will significantly empower our next-generation backend to scale effortlessly under heavy workloads. I have additionally implemented a myriad of comprehensive error handling mechanisms which will ensure that the system remains stable during unexpected outages.

This baseline output contains overly long sentences. It uses filler phrases like "seamlessly refactors" and "leverage a more robust". It nominalises verbs ("utilisation of this new paradigm"). It is incredibly difficult to read quickly.

**The STE-Corrected Output:**

> This pull request changes the database migration scripts to use asynchronous processing. This change allows the backend service to process more concurrent requests. The code now includes new error handling functions. These functions stop the service from crashing when the database connection drops.

The STE version relies on short declarative statements. It uses no contractions and removes the marketing filler entirely. It states exactly what changed and why it matters.

### Example 2: API Authentication Error Messages

Error messages require absolute clarity. A user encountering an error does not want a polite apology or a philosophical explanation of system architecture. They want to know what broke and how to fix it immediately.

**The Baseline AI Output:**

> Oops! It looks like you're trying to access a protected resource, but your authentication token has unfortunately expired or is otherwise invalid. To facilitate a secure environment and ensure your data remains completely safe, we kindly ask that you generate a new token via the developer portal and try your request again.

This baseline uses contractions ("looks like", "you're"). It includes unnecessary apologies and polite filler ("Oops!", "unfortunately", "kindly ask"). It wastes the developer's time with a rolling-window explanation of security policies.

**The STE-Corrected Output:**

> The server rejected your request because the authentication token is invalid or expired. You must use a valid token to access this resource. Open the developer portal. Generate a new authentication token. Send your request again with the new token.

The STE error message is significantly shorter. It contains zero grammatical violations. It provides explicit sequential instructions to the user. It puts the condition before the command.

### Example 3: Kubernetes Deployment Guide

Technical guides generated by default LLMs often bury the actual commands inside massive paragraphs of contextual fluff.

**The Baseline AI Output:**

> Prior to initiating the deployment of the Nginx ingress controller, it is highly recommended to ensure that your Kubernetes cluster is fully operational and healthy. Once you have validated the cluster status, you can proceed to utilise the Helm package manager to effortlessly spin up the required resources. Executing the command below will seamlessly apply the standard configuration to your environment.

This baseline uses banned words ("Prior to", "initiating", "utilise"). It uses phrasal verbs ("spin up"). It uses passive constructions and stacked auxiliaries ("it is highly recommended to ensure").

**The STE-Corrected Output:**

> Check the health of your Kubernetes cluster before you start the deployment. Use the Helm package manager to install the Nginx ingress controller. Run the command below to apply the standard configuration.

The STE version removes the cognitive friction. It uses direct action verbs. It limits each sentence to a single clear instruction.

### Example 4: Architecture Decision Record (ADR)

An ADR must document a technical decision clearly for future engineers. AI models typically inflate these documents with unnecessary justifications.

**The Baseline AI Output:**

> After conducting a comprehensive evaluation of myriad message broker solutions, we have ultimately decided to pivot towards adopting RabbitMQ for our new microservices architecture. The primary driver behind this strategic choice was the fact that RabbitMQ provides battle-tested reliability and a plethora of advanced routing capabilities that will supercharge our data pipelines. Whilst Kafka was also considered, its operational complexity was deemed too high for our current engineering capacity.

This baseline is saturated with marketing adjectives ("battle-tested", "strategic", "comprehensive"). It uses archaic or overly formal terms ("whilst", "plethora", "myriad").

**The STE-Corrected Output:**

> We will use RabbitMQ as the message broker for the new microservices architecture. We chose RabbitMQ because it routes messages reliably and supports our required data pipelines. We evaluated Apache Kafka. We rejected Apache Kafka because it requires more maintenance time than our engineering team can provide.

The STE version breaks the decision down into logical facts. It names the items consistently. It avoids emotional or exaggerated language.

## Summary

We cannot trust LLMs to police their own writing style through vague suggestions. If we want professional, maintainable, and clear documentation, we must enforce a rigid framework.

Simplified Technical English provides the exact mechanical rules that an AI needs to generate high-quality prose. By integrating the STE guidelines into our system prompts, we cure AI slop at the source. This ensures our infrastructure documentation remains as clean, predictable, and reliable as our code.