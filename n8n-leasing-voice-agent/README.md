
We’ve been building voice agents for local businesses for the past 2 months, but always felt the gap with how we actually fit into their workflow. So I tried n8n.

This is the first full n8n flow I put together and I learned A LOT.

# Why missed calls

Voice agents that try to do everything are hard to pull off and even harder for businesses to trust. That’s why I’ve been focusing on simple, repetitive use cases like missed calls.

Leasing offices miss a lot of calls, especially after hours, and many of those turn into lost leads. The thing is, most of them are basic: unit availability, move-in dates, pets, parking, hours (and voice agents are pretty good at this).

# Building the voice agent

I used Alcamine to build the voice agent and deployed it to a phone number (so leasing offices can forward missed calls directly).

* Here's the full prompt I used: [https://alcamine.com/templates/leasing-agent](https://alcamine.com/templates/leasing-agent)
* I used the knowledge base feature to fill it with FAQs and availability for units. Using dummy data here but here's what that looks like: [https://share.cleanshot.com/6dFMSmhz](https://share.cleanshot.com/6dFMSmhz)

# Building the n8n workflow

The n8n workflow is straightforward: take the call transcript from the voice agent, extract the name and a short summary (with an n8n agent), output structured JSON, and push it into a CRM.

**Webhook + If Node**

* Webhook listens for completed calls from the voice agent (Alcamine's API).
* The voice agent API responds with a lot of information, so I used an If node to filter down to the right agent and response.

**AI Agent Node (for summarizing and parsing calls)**

Honestly, my favorite feature from n8n. I tried to do this bit with code and an LLM node, but the **AI Agent Node + Structured Output Parser** made it way easier.

The agent does two things:

* Extracts the caller’s name (if they mention it)
* Summarizes the call in a short note for the CRM

Here's the prompt I used for the n8n agent:

    Extract structured JSON from these messages:

    {{ JSON.stringify($json.body.properties.messages) }}

    Context:
    - Input is a stringified JSON array called "messages".
    - Each item has content.role and content.content.
    - Only use caller ("user"/"customer") content. Ignore assistant/system/tool text.

    Return ONE JSON object in this schema (output valid JSON only, no extra keys or text):

    {
      "caller_name": string|null,
      "notes": string|null
    }

    Rules:
    - caller_name:
     - Extract only if the caller states their own name (e.g., “My name is Sarah”, “This is Mike”).
      - If the caller does NOT state a name, output the EXACT string: "No Name Given".
      - Do NOT infer from email/phone. Do NOT use placeholders like “John Doe”, “Unknown”, etc.
      - If multiple names appear, choose the most recent explicit self‑intro. Ignore third‑party names.
    - notes:
      - Write a single short paragraph summarizing why they called.
      - Include key details (property, unit type, move-in timing, pets, parking, etc.) if mentioned.
      - Keep it under 300 characters. No bullets, no line breaks, no system text.

**Syncing with Pipedrive**

Getting the data into the CRM required two steps:

* Create the person/contact
* Create a note using that person’s ID

# Challenges

I originally wanted to build this in HubSpot, but it requires emails to create a contact. There's a few ways we could solve this.

Option 1: Send a short form after the call to capture email + extra details that are easier to type vs say out loud.

Option 2: Build a texting agent to follow up with SMS + quick questions. This could trigger after the call.

I'm leaning towards the second option but feels harder to pull off.
