# Better names for logical sessions

Original thread starter: **Kurtis Van Gent (Google)**

> Goal: Find a clearer name than "session" for the *logical* shared context (distinct from transport/connection-level sessions) discussed in SEP-1442.

---
## September 11, 2025

**6:23 PM – Kurtis Van Gent (Google)**  
One of the things that came up in the discussion with Mike is there is still a bunch of confusion around the word "session" – specifically on "transport sessions" vs "chat sessions". How do folks feel about renaming it to `context/create` and `mcp-context-id` to help fully distinguish it?

**6:26 PM – Kurtis Van Gent (Google)**  
@evalstate open to alternatives 🙂

**6:27 PM – evalstate**  
'gonna sleep on it!

**6:32 PM – Kurtis Van Gent (Google)**  
I asked Gemini and it gave me some decent options:  
![Brainstormed name suggestions (Gemini output)](https://media.discordapp.net/attachments/1415825198065909780/1415827482204045373/image.png?ex=68cbdfd8&is=68ca8e58&hm=90a33b8c71a98a40ada30d8e96928aa24fb0e3f710730d6d00cb778ccf24aa00&=&format=webp&quality=lossless&width=1528&height=1090)

**6:33 PM – (inline continuation)**  
Context, Thread, Scope, and Conversation all seem like reasonable picks. *(edited)*

**6:34 PM – (inline continuation)**  
@Mike Kistler (Microsoft) 🇺🇸 for your input

**6:50 PM – Cliff Hall (Futurescale)**  
Evaluation of candidates:

- **Context** – Sounds great except the term is already used by LLMs (client-managed). Not the same thing, could confuse developers.
- **Thread** – In multi-threaded environments (e.g. Java `Thread`) could confuse developers.
- **Scope** – Used by OAuth, also potentially confusing.
- **Lease** – Feels weak; doesn’t convey managing a shared logical set of data.
- **Conversation** – *Strongest one yet IMO.* Says what session does without saying session.
- **Workspace** – Meh. Implies project creation/manipulation; sometimes true, but mostly it’s just a session/conversation. *(edited)*

**7:00 PM – Kurtis Van Gent (Google)**  
I'm a little worried that "conversation" is pigeonholing the use somewhat but maybe that's a feature and not a bug. *(edited)*

**8:54 PM – Chip (Ignission)**  
From my perspective as an MCP server developer, my assumption was that a session was intended to be per "chat" (that's what the clients call it).

However, when I use ChatGPT, for example, I often branch chats as well. I could see a case where I'm working in a chat and then branch into 2 separate chats but would not want my session to be reset in the new chat. So if a "conversation" could include multiple "chats" then I think conversation is a great option.

I don't think any of the other options are better than "session".

**8:56 PM – Kurtis Van Gent (Google)**  
Slight tangent, but I was actually thinking about this use case the other day — two ideas I thought of were sessions/copy and "subsessions".

**9:28 PM – Chip (Ignission)**  
Yes, it seems like this will be a necessary addition. ChatGPT recently made conversation branching an explicit feature (although you could always do it via the edit button in conversations).  
![Image: Conversation branching UI](https://media.discordapp.net/attachments/1415825198065909780/1415871735181344940/image.png?ex=68cb604f&is=68ca0ecf&hm=2f42573e31ceecfa3c72fb84bb45197e35c5aefb1e0e59bf69edae43af3c2a29&=&format=webp&quality=lossless&width=998&height=316)

**9:32 PM – Geoff (Auth0 @ OKTA)**  
A word that hasn't been used: **Trace** — as in the directing line from start to end. *(edited)*

**9:33 PM – (inline continuation)**  
Not saying I like it better than Chat. I think Chat would resonate with the different personas of people involved in MCP. *(edited)*

---
## September 12, 2025

**10:00 AM – Kurtis Van Gent (Google)**  
It doesn't sound like we have strong consensus on if any alternatives are better. *(edited)*

**10:00 AM – evalstate**  
not promoting it, but **sequence**?

**11:52 AM – Jonathan Hefner**  
I think a new name could be beneficial as a clean break from the existing session concept / API.

I personally like "conversation", and I agree that the pigeonholing is more of a feature than a bug. I think "chat" could also work similarly, though it feels a bit informal.

**11:56 AM – Mike Kistler (Microsoft) 🇺🇸**  
Good topic. I was thinking "scope" but I agree that could be confused with OAuth scopes.

**12:15 PM – Geoff (Auth0 @ OKTA)**  
We need a doc that outlines the different kinds of state, who the audience is for that state and how it would be passed from the owner to the audience(s). The scope of this state should also be included; is it scoped to a connection, to a "conversation" or to a single RPC?

**12:21 PM – Kurtis Van Gent (Google)**  
I think this thread is specifically limited to SEP-1442, which explicitly says sessions are logical (not scoped to a connection or RPC) and can be scoped to whatever the client decides (app, user, chat).

**12:28 PM – Cliff Hall (Futurescale)**  
Seems like in that context, conversation is the loosest-fitting term we've heard so far that fits what's going on. The client is having a conversation with the server. What it's about is up to the client.

**12:36 PM – Kurtis Van Gent (Google)**  
@Peter Alexander (Anthropic) mentioned some concerns with conversation. Anything in the thread changed your mind so far?

**12:37 PM – (inline continuation)**  
(It feels like there's starting to be some agreement on "conversation")

**12:38 PM – evalstate**  
i prefer conversation to context (even if it is less 'technical')

**12:52 PM – Mike Kistler (Microsoft) 🇺🇸**  
My preference would be to continue to use "session" for the current concept in MCP, clarifying it as needed, and use a new term for "logical" scope/conversation/thingy.

**12:53 PM – Peter Alexander (Anthropic)**  
Hmm, but conversation also could be confused with the conversation with the LLM? i.e. I'm having a conversation with Claude, and that conversation/chat has an ID, but that's different from the MCP client-server 1:1 sessions.

I'm still in favor of just continuing with session.

**12:56 PM – Kurtis Van Gent (Google)**  
I think the idea is that it represents that shared context of the conversation to the server.

**12:59 PM – (inline continuation)**  
So it being tied to the conversation with Claude is kind of the point. 👍

**1:08 PM – Peter Alexander (Anthropic)**  
But it's different scope, no? i.e. a single conversation with the LLM involves many separate MCP conversations/sessions?

**1:42 PM – Cliff Hall (Futurescale)**  
Agreed I don't see the need to move away from Session, just pushing it to the data layer and out of the transport.

**2:25 PM – Mike Kistler (Microsoft) 🇺🇸**  
I think the current "session" in the HTTP transport should stay where it is, and we should have a new (similar, but distinct) mechanism at the data layer. Why? Least disruptive to existing implementations.

We should *clarify* what sessions mean in the HTTP transport; many aspects are under-specified. They were included for a valid reason that remains.

**2:26 PM – (inline continuation)**  
I think the purpose of MCP sessions — and why they are only defined for the streamable HTTP transport — is to allow a server using the HTTP transport to provide the same "one client process talking to one dedicated server process" behavior inherent in STDIO.

In STDIO the server is tightly coupled (child process) so all interactions can be considered logically related. The spec does not say which interactions on a session are logically related. That implies *they all are* — just as they would be in STDIO. 💯

**2:33 PM – Cliff Hall (Futurescale)**  
The reason for moving session up from the transport layer is to avoid needing a one client to one server relationship. If the session was a data layer construct, it could be persisted and you could horizontally scale to thousands of servers behind a load balancer. Transport-level sessions hamstring scaling. *(edited)*

**3:15 PM – Kurtis Van Gent (Google)**  
I think there is some agreement that there is:
  1. a session concept at the transport layer (representing a "client" talking to a server)
  2. a session concept at the logical layer (representing a chat or conversation or some shared context between requests)

**3:16 PM – (inline continuation)**  
SEP-1442 tries to say:
  1. let's get rid of (1) because it's problematic to serve MCP at scale
  2. let's introduce a proper mechanism for covering (2)

**3:19 PM – (inline continuation)**  
(And this thread is trying to figure out what to name (2)) 👍

**3:54 PM – Mike Kistler (Microsoft) 🇺🇸**  
> The reason for moving session up from the transport layer is to avoid needing a one client to one server relationship.

This is already supported with the current spec. A server that wants to avoid this never sends a `sessionId` back. A client that wants to avoid it never includes one in requests.

**3:55 PM – (inline continuation)**  
> SEP-1442 tries to say: let's get rid of 1 because it's problematic to server MCP at scale

I think it is unnecessary to get rid of (1) because any server that wants to can simply choose not to implement sessions.

**4:04 PM – Cliff Hall (Futurescale)**  
> This is already supported… (statelessness argument)

That's statelessness. We're talking about having *stateful* sessions, just not having them tied to the transport.

**4:41 PM – Mike Kistler (Microsoft) 🇺🇸**  
After another 1:1 with Kurtis: his concern isn't really about the Streamable HTTP transport (which can already be stateless). It's about the STDIO transport, which has no session concept and thus no way for the server to "opt out" of stateful interactions. Is that what everyone else is concerned about?

**5:13 PM – Geoff (Auth0 @ OKTA)**  
It's not my angle but if we can support session multiplexing in STDIO, the protocol would probably be improved for it.

**5:16 PM – Kurtis Van Gent (Google)**  
Maybe more broadly, the transport needs to be an implementation detail for the protocol. 👍

---
*Reconstructed faithfully from raw exported HTML/Discord thread. Reactions kept only where they added emphasis (👍, 💯). No substantive wording changed except minor punctuation normalization.*
