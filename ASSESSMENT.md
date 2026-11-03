# Senior AI Cloud Engineer: take-home design exercise

**Time:** about 6 hours of work. **Due:** 7 calendar days after you receive this.
**Then:** a 60-minute session where you walk us through your design and we go deeper, live.

## Rules

- You may use any AI tools you like. In the follow-up session you won't, and we will ask you to
  explain, extend and change any part of your design on the spot. Write only what you can defend.
  The session also includes a short review of a design excerpt we'll show you on screen.
- Tell us how you used AI (section 11).
- This scenario is hypothetical. It is realistic on purpose, but it does not describe ASU's
  systems, and nothing in it implies a gap in them. We will not use your design in production.
- Don't include anything confidential from a current or past employer.
- **Clarifying questions:** you may send up to three, as a reply to the message that invited you,
  within the first 72 hours. We send every question and its answer, without names, to everyone taking this
  exercise, so all candidates work from the same information. If you'd rather not ask, state
  your assumption and move on. Both are fine.
- If you need an accommodation or more time, tell us. It won't count against you.
- Please don't post your submission publicly.

## What we're grading

Decisions, not volume. This brief contains more than anyone can address well in 4,000 words, and
some of its requirements conflict with each other. Noticing that, deciding, and telling us what
you cut and why is most of the exercise. A shorter document with clear choices beats a long one
that covers everything at the same depth. We stop reading at the word limit.

## The situation

A large public university runs a model-agnostic AI platform on AWS: 150,000 students, 9,000
instructors and TAs, 600 academic advisors. People use it for chat and agents. They sign in with
university SSO and choose a model for each conversation. The platform is an MCP client and speaks
the 2026-07-28 MCP revision.

You are the senior engineer asked to design the **Tool Gateway**: the single path by which agents
on the platform call tools that reach university systems. Build on AWS. Phase 1 is one quarter,
and you have two mid-level engineers.

### Systems the gateway must connect in phase 1

| System | What agents need | How it can be reached | Limits and quirks |
|---|---|---|---|
| **LMS** (SaaS) | List my courses; read pages, files and modules; read discussion posts and submissions (instructors and TAs); read the in-term gradebook (instructors); post announcements; send inbox messages | Per-user OAuth 2.0 authorization-code grant. Access tokens last 1 hour. Refresh tokens rotate on every use, and reusing an old one revokes the user's whole grant. | Per user token: 60 requests/minute sustained, burst of 30. Institution-wide: 1,000 requests/second across all tokens; above that, `429` with `Retry-After`. A tool call averages 1.4 LMS requests (pagination). Permissions are per course, and can be per section, module or date. |
| **Student information system** (on premises) | Section rosters, instructor and advisor assignments, academic standing, final grades, privacy holds | REST over private connectivity to AWS. No OAuth: one service credential that can read every record. | At most 50 concurrent requests from the platform. Latency p50 150 ms, p95 400 ms, p99 2 s. Down nightly 01:00–03:00. A change feed publishes enrollment, instructor-assignment and advisor-assignment changes, typically within 30 seconds; during registration peaks it can lag up to 15 minutes. |
| **Workspace documents** (Google Workspace or Microsoft 365: pick one) | Read the user's documents; create documents in a folder the user picks | Per-user OAuth | Per-user limits you won't hit; no institution-wide cap. |
| **Publisher's MCP server** (third-party SaaS) | Textbook search; practice-quiz generation | Remote, Streamable HTTP, 2025-11-25 revision. Multi-tenant, run by the vendor. | Contract: 300 requests/second, and the license covers 40 million calls a month. The vendor can change its tool list and descriptions at any time. Its integration guide asks you to forward each user's LMS access token "so we can look up their enrolled courses." |
| **Department MCP servers** | 25 departments want to build their own: lab booking, library, tutoring, advising notes and so on | Small teams writing Python or TypeScript, mostly without security specialists | Some will stay on the 2025-11-25 revision for at least a year. |

### Identity

- The university IdP uses OIDC. Access tokens issued to the platform are 60-minute JWTs, and the
  IdP team won't shorten them. The groups claim reflects membership at sign-in.
- For account disables, the IdP team will give you **either** a webhook that fires within a
  minute **or** a token-introspection endpoint limited to 20 requests/second. Not both.
- The university's authorization server supports OAuth 2.1, resource indicators (RFC 8707) and
  token exchange (RFC 8693).

### Load

- Peak (finals-week evenings): 30,000 concurrent active conversations, 3 agent turns per
  conversation per minute, 2.5 tool calls per turn on average (range 0–8).
- Tool-call mix: LMS 45%, workspace 20%, publisher 15%, student information system 10%,
  department servers 10%.
- Across a month, average load is 12% of peak.

### Requirements

1. **Overhead:** the gateway adds at most 100 ms at p95 to a tool call, not counting the
   downstream system's own time.
2. **Availability:** 99.9% monthly for the gateway, in one AWS region. Two engineers share
   on-call. State the RTO and RPO for anything stateful.
3. **Revocation:** when a student drops a section, an instructor is removed from one, an advisor
   is reassigned, or an account is disabled, access through the gateway stops within 5 minutes,
   including in conversations that are already open.
4. **Writes:** posting, sending or creating anything needs the user's explicit confirmation of
   that exact action in the UI, and a model must not be able to complete a write without it.
   Every write happens at most once, even when the client retries, the model repeats the call, or
   the connection drops after the downstream system has committed.
5. **Long-running work:** "Build a study guide from all my course materials" takes 3–12 minutes
   and 200+ LMS reads. The user may close the browser. The job must finish, and the user must be
   able to get the result.
6. **Data classes:** *Sensitive* means grades (from either system), academic standing, privacy
   holds, and anything a student wrote (submissions, discussion posts). Sensitive data may reach
   only models approved for it, which today means models on Amazon Bedrock. Users choose the
   model; the gateway is where this rule is enforced.
7. **Untrusted content:** discussion posts, submissions, uploaded files and anything the
   publisher returns are attacker-controlled. Assume a student will try to get an instructor's
   agent to leak classmates' grades or post on their behalf.
8. **Privacy holds:** no tool may reveal the name or enrollment of a student with a privacy hold
   to other students.
9. **Audit:**
   - The registrar requires every agent access to Sensitive data to be attributable to the
     person, the conversation, the tool and the records touched, and reconstructable for 5 years.
   - The privacy office requires that no grades or student-written content appear in
     application logs.
   - The security office wants full request and response capture for 30 days, for incident
     response.
10. **Third-party servers:** define how a third-party MCP server gets approved, what you pin, and
    what happens when its tools change.
11. **Department template:** what the 25 teams get, and what they cannot opt out of.
12. **Cost:** the gateway's running cost is at most $15,000 a month at this load, not counting
    model inference or the downstream systems' own costs. Show the estimate.
13. **Infrastructure as code:** everything in Terraform or OpenTofu, deployed by CI/CD.
14. **Phase 2:** external MCP clients (a desktop AI app, for example) will connect to the gateway
    directly. Phase 1 doesn't build this but must not rule it out.
15. **Chat UI:** the platform renders model output as Markdown, including links and images. The
    platform team owns the UI and wants your requirements for anything that displays tool
    results or asks the user to confirm something.

### Requests from stakeholders

- **Product:** "Show every tool to every user, so the agent can figure out what's useful."
- **Faculty senate:** "Instructors want an auto-grade button: the agent grades the submissions
  and posts the grades to the gradebook."
- **The publisher:** forward the user's LMS token (see the systems table).
- **A college:** "We built an advising agent that speaks A2A. Can the platform call it as a tool?"

## What to send us

One document, PDF or Markdown. **4,000 words maximum** for sections 1–8, 10 and 11, counting
text inside tables; diagrams don't count. Code goes in an appendix that doesn't count toward the
4,000: the §8 tool definition and hook (up to 80 lines) and the §9 Terraform (up to 150 lines).
We stop reading at each limit. Use these section numbers.

1. **Summary** (400 words maximum). The design in five bullets, the three biggest risks, and what
   ships in phase 1 versus later.
2. **Architecture.** Components, trust boundaries, and every token: who issues it, its audience,
   its lifetime, where it is stored, and who can read it.
3. **One flow, end to end.** An instructor teaches sections 1 and 2 of a course and is using an
   approved model. They ask: *"Which of my section 2 students are below a C right now? Draft each
   of them a short check-in message in the LMS inbox, and let me review before anything is
   sent."* One student in section 2 has a privacy hold. A discussion post the agent read earlier
   in the conversation contains an injected instruction. Show every authorization decision,
   token, data-class check, confirmation and write, and what goes into the audit trail. For the
   step where the instructor confirms the send, include the MCP messages as JSON: the request,
   what the gateway returns, and the request that completes the send, on the protocol revision
   that hop actually speaks.
4. **Authorization.** The decision model, where it is enforced, its data sources, and the
   freshness math that meets requirement 3 within requirement 1. Use an advisor asking about
   their advisees as a second worked example. Say what happens when each dependency is slow,
   stale or down.
5. **Capacity and cost.** The load your design carries and its monthly cost, with the arithmetic.
6. **Threat model.** The specific attack paths for requirement 7 and for the publisher's server,
   the control for each, and the risk that remains.
7. **Conflicts and decisions.** Every conflict you found in this brief, how you resolved it, and
   whose agreement you would need. Then the three decisions you are least sure of, and what
   evidence would make you reverse each one.
8. **Department template.** What is in it, with one example tool definition (input and output
   schema, annotations) and the authorization hook. Include how a server is tested before it is
   published, including how you would know models pick the right tool.
9. **Terraform** (appendix, at most 150 lines). For the single most security-sensitive component
   of your design. Tell us why you picked it.
10. **Phasing.** What you and two mid-level engineers ship in one quarter, and what you cut.
11. **AI use** (250 words maximum). Which tools you used and for what. Then up to three places
    where an AI's output was wrong or where you overruled it: quote what it said, what was wrong,
    and what you did instead. In the session we'll ask you to walk us through one of them. If you
    didn't use AI, say so and skip the rest.

After you submit, we'll schedule the 60-minute session. Have your document open; we'll share
our screen with it too.
