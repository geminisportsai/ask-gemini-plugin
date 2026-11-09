# Gemini Sports for AI Assistants: Privacy Policy

Last updated: November 9, 2026

This policy covers the Gemini Sports plugin and MCP connector for AI assistants, including Claude and ChatGPT (together, the "Connector"), provided by Gemini Sports Inc. It explains what data the Connector handles, who else receives it, how long we keep it, and what choices you have. For how the Connector works and how to install it, see [Gemini Sports for Claude](README.md).

## 1. Who the Connector Is For

Gemini Sports customer organizations are private, and access is by invitation only. The Connector works only for people who already have an active Gemini Sports club account. It is meant for professional use and is not directed at anyone under 18.

## 2. What We Receive

We do not receive your conversation with your AI assistant. We receive only the tool calls the assistant makes to the Connector: the name of the tool and the inputs the assistant sends with it, such as a player name or a watchlist ID. Our server checks who you are, runs the request against your organization's data, and returns the result to the assistant.

## 3. Data the Connector Processes

- **Your phone number** – You sign in with your phone number and a one-time code sent to you by SMS. We use your phone number to verify your identity and send that code.
- **Sign-in (OAuth) tokens** – After you sign in, we issue OAuth tokens. Your AI assistant stores them and presents them with each tool call so we can confirm it is you.
- **Your organization's data** – The data the Connector reads or writes for you, such as watchlists, pipeline boards, scout reports and notes, contracts and squad-planning scenarios. The Connector only reaches your own organization's data. Its tools apply the same role permissions as the Gemini Sports app, except stat lookups, which run as read-only queries limited to your club's data.
- **Player reference data** – Football data about players, such as profiles, statistics and transfer history.
- **Request logs** – Records of requests to the Connector, kept to run, secure and troubleshoot the service. Identifiers in these logs are masked.

## 4. What the Connector Can Change

The Connector can make only these changes in your Gemini Sports workspace:

- Add a player to, or remove one from, a watchlist or pipeline board
- Create a watchlist
- Create a private pipeline board
- Delete a conversation thread

## 5. Who Else Receives Data

- **Your AI assistant's provider** – Tool results go to the assistant you use, inside your own session, so that it can answer you: Anthropic for Claude, or OpenAI for ChatGPT. That company handles them under your agreement with it. We do not control that processing.
- **Amazon Web Services** – Hosting in the United States, sign-in, and delivery of SMS sign-in codes.
- **Sentry** – Error monitoring.

Connector data is stored in the United States. If you use the Connector from outside the United States, your data is transferred to the United States.

## 6. How Long We Keep Data

- **Account data** – Kept while your Gemini Sports account is active.
- **Organization data** – Governed by your club's agreement with Gemini Sports. Data you create or change through the Connector lives in Gemini Sports like any other data in your workspace.
- **Request logs** – Kept for up to 12 months, with identifiers masked.

## 7. Your Choices

You can disconnect the Connector in your AI assistant at any time, which stops further tool calls. To access, correct or delete your personal data, contact us at the address below.

## 8. Terms

Use of the Connector is also governed by our [terms of service](TERMS.md).

## 9. Contact

For questions about the Connector or this policy, contact [support@geminisports.ai](mailto:support@geminisports.ai).
