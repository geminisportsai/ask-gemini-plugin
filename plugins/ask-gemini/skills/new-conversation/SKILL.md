---
name: new-conversation
description: >
  Acknowledge a request to start a fresh conversation or reset the current one.
  Use when the user wants to clear context and begin a new topic.
---

# New Conversation

Use this skill when the user wants to start fresh or reset the current conversation
(e.g. "start a new conversation", "clear the chat and start over", "reset this thread").

## What to do

Respond with a short, friendly confirmation that they can start fresh, and invite
their next question. That is the entire task — do not call any tool.

Example:

> Sure — let's start fresh. What would you like to know?

## Why there is no tool call

Starting a new conversation is a **client-side action**. The app opens a new session
for the next message, so the prior context naturally stops carrying over. You do **not**
have access to the current thread's identifier and there is no tool you can call to
delete it — attempting one only produces an error. Never ask the user for a thread ID
and never claim you deleted or cleared server-side data. Simply acknowledge the reset.
