# LLM extraction pipeline

_Backend-leaning. Focus: handling high latency and unreliable LLM APIs._

## Prompt

> We build a research assistant for scientific literature. A core feature is **extraction**: a user selects a set of papers (up to about 1,000) and defines columns in plain language, like "sample size", "intervention" or "primary outcome". For every paper × column cell, we call an LLM to read the paper and extract the answer, with a supporting quote.
>
> The LLM APIs we depend on are slow and unreliable: responses take anywhere from a few seconds to over a minute, and a meaningful fraction of calls fail or get rate limited.
>
> Design the backend system that runs these extractions and gets results to the user.

Target length: 60–90 minutes. Clarifying questions are answered in [interviewer-notes.md](interviewer-notes.md). Try to ask them yourself before reading it.

## Requirements and scope

## Estimates and constraints

## API and data model

## High-level architecture

## Deep dive

## Scaling, failure modes and wrap-up
