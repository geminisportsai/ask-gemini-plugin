# Gemini Sports for Claude: Privacy Policy

Last updated: November 7, 2026

This policy covers the Gemini Sports plugin for Claude and the Gemini Sports MCP connector it uses (together, the "Connector"), provided by Gemini Sports Inc. It explains what data the Connector handles, who else receives it, how long we keep it, and what choices you have. For how the Connector works and how to install it, see [Gemini Sports for Claude](README.md).

## 1. Who the Connector Is For

Gemini Sports customer organizations are private, and access is by invitation only. The Connector works only for people who already have an active Gemini Sports club account. It is meant for professional use and is not directed at anyone under 18.

## 2. What We Receive

We do not receive your Claude conversation. We receive only the tool calls Claude makes to the Connector: the name of the tool and the inputs Claude sends with it, such as a player name or a watchlist ID. Our server checks who you are, runs the request against your organization's data, and returns the result to Claude.

## 3. Data the Connector Processes

- **Your phone number** – You sign in with your phone number and a one-time code sent to you by SMS. We use your phone number to verify your identity and send that code.
- **Sign-in (OAuth) tokens** – After you sign in, we issue OAuth tokens. Claude stores them and presents them with each tool call so we can confirm it is you.
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

- **Anthropic** – Tool results go to Anthropic's Claude inside your own Claude session, so that Claude can answer you. Anthropic handles them under your agreement with Anthropic. We do not control that processing.
- **Amazon Web Services** – Hosting in the United States, sign-in, and delivery of SMS sign-in codes.
- **Sentry** – Error monitoring.

Connector data is stored in the United States. If you use the Connector from outside the United States, your data is transferred to the United States.

## 6. How Long We Keep Data

- **Account data** – Kept while your Gemini Sports account is active.
- **Organization data** – Governed by your club's agreement with Gemini Sports. Data you create or change through the Connector lives in Gemini Sports like any other data in your workspace.
- **Request logs** – Kept for up to 12 months, with identifiers masked.

## 7. Your Choices

You can disconnect the Connector in Claude at any time, which stops further tool calls. To access, correct or delete your personal data, contact us at the address below.

## 8. Contact

For questions about the Connector or this policy, contact [cody@geminisports.ai](mailto:cody@geminisports.ai).
