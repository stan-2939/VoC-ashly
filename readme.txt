================================================================================
  PURCHASE ORDER AGENT - FOUR GOVERNANCE PILLARS ON GOOGLE CLOUD
  SPEAKER SCRIPT

  Deck : PO_Agent_Governance_Deck.pptx  (72 slides)
  Notes: PO_Agent_Governance.md         (full technical detail)

  Full run  : ~85 minutes + 15 min Q&A
  Short cut : ~30 minutes  -> present only slides marked [*]
================================================================================


HOW TO USE THIS SCRIPT
--------------------------------------------------------------------------------
Every slide has a SAY block. That is roughly what to say out loud, in plain
spoken English. Do not read it word for word - read it twice beforehand, then
speak from the shape of it. Short sentences. Pause at the full stops.

Some slides also have:
   POINT AT ...  the one thing to physically point to on the slide
   IF ASKED ...  the answer to the question that slide usually provokes
   TRANSITION .. the sentence that carries you into the next slide

[*] means "keep this slide in the 30-minute version".


THE FIVE SENTENCES YOU MUST LAND
--------------------------------------------------------------------------------
If you run out of time and say nothing else, say these five things.

  1. This agent spends real money in SAP, so a mistake is not a bad answer -
     it is a wrong purchase order.
  2. The model plans. Code enforces. Never the other way round.
  3. We carry two IDs: one for debugging, one for audit. The audit one never
     expires.
  4. A human approval is cryptographically tied to one exact payload, so
     "approve one thousand dollars, execute one million" is impossible.
  5. Prompt injection is not fully solvable, so we designed the system so that
     a successful injection is not enough to cause harm - and we test that
     claim on every single deploy.


TIMING PLAN (full run)
--------------------------------------------------------------------------------
   Slides  1 - 7    Setup and framing            10 min
   Slides  8 - 19   Pillar 1  Observability      15 min
   Slides 20 - 33   Pillar 2  RAG and Memory     15 min
   Slides 34 - 50   Pillar 3  Autonomy and HITL  20 min
   Slides 51 - 62   Pillar 4  Testing            15 min
   Slides 63 - 72   Synthesis and close          10 min

If you are running late, cut slides 11, 14, 25, 29, 33, 43, 48, 59, 68.
Never cut slides 7, 28, 40, 46, 53. Those five are the argument.


ONE HOUSEKEEPING NOTE - SAY THIS EARLY
--------------------------------------------------------------------------------
Some acronyms in the case study are Ingram-internal and I could not expand
them - IMPO, orp_tool, MCNapitool, LOPapiTool, GNE Mail. I have marked those
as assumed in the written notes. The design does not depend on the expansions.
It depends only on two things per tool: does it write somewhere, and how much
damage can it do. Correct me on the names and nothing in the design changes.



================================================================================
  PART 0 - SETUP AND FRAMING
================================================================================

--------------------------------------------------------------------------------
SLIDE 1  [*]  TITLE
--------------------------------------------------------------------------------
SAY:
  Thanks for the time. The brief was to take the Purchase Order Agent and say
  how we would put it into production properly on Google Cloud - across four
  governance areas. So that is what this is. It is a design, not a survey. I
  will name specific services, specific database tables, specific code hooks,
  and I will tell you where I think the architecture as drawn has holes.
  I will keep stopping for questions, so jump in.

TRANSITION:
  Let me start with why this needs governance at all.


--------------------------------------------------------------------------------
SLIDE 2  [*]  AGENDA
--------------------------------------------------------------------------------
SAY:
  Quick map. First, understanding the system we were given - including seven
  gaps I found in it. Then one section per pillar, three or four slides each.
  Then I pull it all together: one architecture diagram, the full service list,
  and the five integration points that actually matter. Then a phased plan,
  the trade-offs, and the risks.

TRANSITION:
  So. What are we actually governing here.


--------------------------------------------------------------------------------
SLIDE 3  [*]  WHAT WE ARE ACTUALLY GOVERNING
--------------------------------------------------------------------------------
SAY:
  This is not a chatbot. This agent releases purchase orders and rejects
  orders. It is an economic actor. If it gets something wrong, that is not a
  bad answer you regenerate - it is real money, or a supplier relationship, or
  an audit finding.
  Second thing. There are two ways in. A human talking to the frontend agent,
  and a scheduled batch job with nobody watching. Those are different risk
  profiles, and I will argue the unattended one should be allowed to do less,
  not more.
  Third thing, and this is the one that shapes half the design: untrusted text
  reaches the model on purpose. Vendor PDFs. Vendor portal HTML. Free text
  fields in SAP. Email bodies. Every one of those is a way for someone outside
  the company to put words in front of our model.

POINT AT: card 04.


--------------------------------------------------------------------------------
SLIDE 4  [*]  THE ARCHITECTURE AS GIVEN
--------------------------------------------------------------------------------
SAY:
  This is the diagram from the case study, redrawn. Scheduler triggers the
  backend agent. The backend is the ADK orchestrator and it owns two
  sub-agents - one for SAP, one for the vendor portal. Separately there is a
  frontend agent with eight tools that a human talks to. Both share one
  AlloyDB. Telemetry goes to Datadog, notifications go out by mail.
  Now, the useful thing I did here was sort the tools by whether they write.
  Six of them only read. Four of them write.

POINT AT: the two lines at the bottom of the slide.

SAY (continue):
  Those four - releasing an order, rejecting an order, submitting a vendor
  form, and the LOP tool - are the entire reason this presentation exists.
  Everything I am about to describe is about those four.


--------------------------------------------------------------------------------
SLIDE 5  [*]  COMPONENT ROLES
--------------------------------------------------------------------------------
SAY:
  Same diagram, read as a risk surface instead of a component list. Two rows
  to notice. The backend orchestrator is where decisions get made, so that is
  where budgets and approval checks have to live. And the SAP agent has the
  biggest blast radius, because it writes the system of record - so every call
  it makes has to be safe to retry.
  One more. The vendor portal agent talks to a third party we do not control.
  Whatever comes back from it is untrusted input, exactly like something a
  stranger typed.

TRANSITION:
  Which brings me to the gaps.


--------------------------------------------------------------------------------
SLIDE 6  [*]  SEVEN GAPS IN THE AS-DRAWN ARCHITECTURE
--------------------------------------------------------------------------------
SAY:
  I want to be direct about this, because it is where the design comes from.
  There are seven things missing from the diagram as drawn.
  The big one is number one. There is no human-in-the-loop machinery at all.
  No approval store, no way to pause, no way to resume, no screen for an
  approver. And yet the agent releases purchase orders. That gap is most of
  pillar three.
  Number two, there is no guardrail layer - user input goes straight to the
  model. Number three, Datadog is the only place telemetry lands, so we lose
  native log-to-trace correlation inside Google Cloud. Number four, there is
  no ingestion pipeline, even though the agent clearly needs vendor terms and
  internal procedures. Number five, the frontend looks directly exposed to the
  internet. Number six, the scheduler hits the backend with no throttle and no
  deduplication. Number seven, one shared database credential for both agents.
  Naming these first is what gives the rest of the deck its shape.

IF ASKED "are you criticising the existing design?":
  No - it is a solution sketch, and it is a good one for showing intent. These
  are the things you would add on the way to production. That is the job I was
  given.


--------------------------------------------------------------------------------
SLIDE 7  [*]  THE ONE PRINCIPLE EVERYTHING FOLLOWS   << DO NOT CUT >>
--------------------------------------------------------------------------------
SAY:
  If you remember one slide, remember this one.
  The model is a planner. It is never an enforcer.
  Every safety property in this design is enforced by something that cannot
  change its mind: deterministic code, a database constraint, an IAM
  permission, or a cryptographic key. Never by a sentence in a prompt.
  Four examples along the bottom. The budget is checked in code before the
  model is called. The approval is checked in code before the tool is called.
  Isolation is enforced by the database, so a forgotten filter returns zero
  rows instead of somebody else's data. And exactly-once is enforced by a
  unique constraint, so a duplicate release is not possible.

PAUSE. Then:
  Here is what that buys us. A completely jailbroken PO agent still cannot
  release a two million dollar purchase order. Guardrails reduce probability.
  Architecture caps impact. I will keep coming back to that distinction.



================================================================================
  PILLAR 1 - RUNTIME GUARDRAILS AND OBSERVABILITY
================================================================================

--------------------------------------------------------------------------------
SLIDE 8  DIVIDER - PILLAR 1
--------------------------------------------------------------------------------
SAY:
  Pillar one. The goal in one sentence: for any action the agent took, at any
  point in the past, I can reconstruct what it did, why it did it, what it
  cost - and prove it never spiralled. Three criteria on the right.


--------------------------------------------------------------------------------
SLIDE 9  [*]  UNIFIED TRACE IDS - THE CONCEPT
--------------------------------------------------------------------------------
SAY:
  One agent turn is not one operation. It fans out into dozens. One request,
  a retrieval query, three model calls, five tool calls, a couple of retries,
  an SAP call, a vendor call, four database writes, an email. Different
  services, different machines, different log files.
  A trace is the whole thing end to end. A span is one piece of work inside
  it, and spans nest into a tree. The industry standard for passing this
  between services is a header called traceparent, and there is a companion
  header called baggage where we put business keys like the PO number.
  And here is the bit most designs get wrong. You need two identifiers, not
  one.


--------------------------------------------------------------------------------
SLIDE 10  TWO IDS, TWO JOBS
--------------------------------------------------------------------------------
SAY:
  On the left, the trace ID. That is technical. OpenTelemetry mints it, it is
  for latency waterfalls and debugging, and in Cloud Trace it disappears after
  thirty days by default.
  On the right, what I am calling the run ID. That is a business identifier.
  We mint it once, at the first user turn, and it goes on everything - every
  span, every log line, every database row, the idempotency key we send to
  SAP, and the footer of the approval email.
  The reason is simple. Traces expire. Audit obligations do not. If someone
  asks about a purchase order from fourteen months ago, the trace is long
  gone. The run ID is still there.
  One practical tip - store the trace ID on the run record. Then you can pivot
  either direction while the trace still exists.


--------------------------------------------------------------------------------
SLIDE 11  UNIFIED TRACE IDS ON GCP     (cuttable)
--------------------------------------------------------------------------------
SAY:
  Concretely on Google Cloud. Each Cloud Run service runs the OpenTelemetry
  SDK in the app, and an OpenTelemetry Collector as a sidecar container next
  to it. The app sends to localhost, the sidecar fans out to two places -
  Cloud Trace and Datadog. One instrumentation, two destinations, so we get
  Datadog for unified ops and Cloud Trace for native correlation without
  paying twice in engineering effort.
  Logs go to standard output as JSON with the trace fields attached, Cloud
  Logging picks them up, and the log router splits them three ways: BigQuery
  for analysis, Datadog for ops, and a locked bucket for compliance.
  Bottom box is the gotchas. I will call out two. Cloud Run injects its own
  trace header, so you need a composite propagator or your load balancer span
  and your app span end up in two separate traces. And in Python, trace
  context is a context variable - if you spawn a task without carrying it
  across, the spans silently orphan. That is the number one cause of broken
  agent traces and it is very annoying to debug.


--------------------------------------------------------------------------------
SLIDE 12  SAMPLING
--------------------------------------------------------------------------------
SAY:
  Sampling deserves a slide because this is where cost and audit collide.
  Recording everything is expensive. But sampling blindly - say, keep one in
  five - will eventually throw away the exact trace an auditor asks about.
  So: rule-based sampling. Keep one hundred percent of anything that writes,
  anything that errored, anything that hit an approval, anything that tripped
  a guardrail, anything that went over budget. Sample the chatty read-only
  conversation at twenty percent.
  The rule is: never sample away the traces you will be asked about.


--------------------------------------------------------------------------------
SLIDE 13  [*]  SEMANTIC TRACEABILITY
--------------------------------------------------------------------------------
SAY:
  Normal monitoring tells you that a call happened, how long it took, and
  whether it failed. For an agent that spends money, that is not enough,
  because it cannot tell you why the agent decided to make that call. And why
  is the entire audit question.
  So we capture more, per step. Who and which run. Which model and which
  version, and which prompt template version. What we retrieved - the actual
  chunk IDs and their scores. The reasoning the model gave, and the plan.
  The tool call, its arguments, and the hash of those arguments. Which policy
  version was in force. And the outcome, including the document number SAP
  gave back.
  That last one matters more than it looks. It is the link between our world
  and the system of record.


--------------------------------------------------------------------------------
SLIDE 14  TWO PATTERNS THAT MAKE IT WORK    (cuttable)
--------------------------------------------------------------------------------
SAY:
  Two practical patterns, because the obvious version of this breaks.
  First, trace pointers. You do not want a two-hundred-kilobyte prompt inside
  a span - trace backends truncate it and it slows everything down. So write
  the full payload to Cloud Storage as an immutable object, and put only the
  path and a hash into the span. Spans stay small. Evidence stays complete.
  Second, hash before you redact. We run data-loss-prevention redaction before
  anything leaves for Datadog or BigQuery. But we compute the hash of the
  original first. That way we can prove "this is exactly what the model saw"
  without storing the personal data itself.


--------------------------------------------------------------------------------
SLIDE 15  WHERE THE HOOKS GO
--------------------------------------------------------------------------------
SAY:
  Here is the implementation answer, and it is nicer than you would expect.
  You do not sprinkle logging through your business logic. ADK gives you six
  callbacks around the agent lifecycle, and you install your instrumentation
  once, centrally, in those.
  And notice the three items in bold. The same hooks that emit telemetry are
  where the enforcement lives. Budget checks go in before-model. Output
  guardrails go in after-model. And the approval check goes in before-tool.
  That is not a coincidence - it is the design. One chokepoint, used for both
  observing and enforcing.


--------------------------------------------------------------------------------
SLIDE 16  THE ACCEPTANCE TEST FOR PILLAR 1
--------------------------------------------------------------------------------
SAY:
  Let me make this concrete instead of abstract. A compliance officer walks in
  and asks: why was purchase order four-five-oh-one-two-three-four rejected on
  the twelfth of August. You have two minutes.
  Here is what you must be able to produce. The user's exact words. The policy
  paragraph the agent retrieved, with its source document and its score. The
  prompt template version. The model and its settings. The reasoning it gave.
  The policy version that triggered the approval. Who approved it and what
  was on their screen when they did. And the SAP document that came out.
  If you can answer that from one query, pillar one is done. If you are opening
  three different tools and guessing, it is not.


--------------------------------------------------------------------------------
SLIDE 17  [*]  COST AND LOOP CIRCUIT BREAKERS
--------------------------------------------------------------------------------
SAY:
  Two failure modes here that do not exist in normal software but are routine
  in agents.
  Loops. Tool fails, agent re-plans, calls the same tool, fails the same way,
  forever. Or two sub-agents politely delegating to each other. The cost goes
  up, progress is zero, and - worse - you might be hammering SAP or a vendor
  portal thousands of times.
  And denial of wallet. Unbounded token spend. Sometimes accidental, like a
  four-hundred-page PDF pasted into context and then retried forty times.
  Sometimes deliberate.
  The pattern we borrow is the circuit breaker from microservices. Closed,
  traffic flows. Open, we fail fast without calling anything. Half-open, we
  try one request to see if it recovered.
  And we apply it in two directions: protect the budget from the agent, and
  protect SAP and the vendor portals from the agent.


--------------------------------------------------------------------------------
SLIDE 18  SIX LAYERS OF DEFENCE
--------------------------------------------------------------------------------
SAY:
  Six layers, because any one control will eventually be misconfigured.
  Layer one, a budget envelope per run - caps on model calls, tool calls,
  tokens, dollars, wall-clock time, and loop iterations. ADK gives you two of
  those for free.
  Layer two, three different loop detectors. Same tool with the same arguments
  three times. Or the world is not changing but the cost is - I hash the
  observable state and watch it not move. Or the plan itself is just being
  restated, which you catch by comparing embeddings of consecutive plans.
  Layer three, and this one bites people: Cloud Run scales out, so a counter
  in memory lies to you. Counters go in Redis so they are shared. And if Redis
  is down, we fail closed on writes - a write we cannot meter is a write we
  cannot bound.
  Layer four protects the systems downstream. Layer five is the money
  backstop, next slide. Layer six is just cost engineering - cheap model for
  extraction, expensive model only for planning, rerank instead of stuffing
  fifty chunks into context, and put deterministic business rules in SQL where
  they belong instead of asking a model to do arithmetic.


--------------------------------------------------------------------------------
SLIDE 19  THE FINANCIAL KILL SWITCH
--------------------------------------------------------------------------------
SAY:
  This exists because every technical cap I just described can be set wrong.
  Money cannot be set wrong.
  Cloud Billing raises budget alerts at fifty, eighty and a hundred percent.
  Those go to Pub/Sub, which triggers a small service that does three things:
  flips a flag so the agent drops to read-only, tightens the model quota, and
  pages someone.
  And then attribution. Label everything, export billing to BigQuery, and join
  it to the run table on the run ID. Which lets you report the number in the
  middle of the slide - cost per purchase order successfully and correctly
  processed. Not cost per token. Cost per token is not a business metric.
  Last thing: monitor the breakers themselves. If loop-breaker trips per
  thousand runs starts climbing, that is not users behaving badly. That is a
  prompt regression, and you want to know within hours.



================================================================================
  PILLAR 2 - RAG AND MEMORY GOVERNANCE
================================================================================

--------------------------------------------------------------------------------
SLIDE 20  DIVIDER - PILLAR 2
--------------------------------------------------------------------------------
SAY:
  Pillar two. Everything the agent knows is clean, scoped correctly to whoever
  is asking, and expires when it should.


--------------------------------------------------------------------------------
SLIDE 21  [*]  WHY THIS IS A SECURITY PILLAR
--------------------------------------------------------------------------------
SAY:
  I want to reframe this one, because people file it under quality and it
  belongs under security.
  Retrieved content becomes prompt content. And prompt content is
  indistinguishable from instructions. So if an attacker can get text into our
  vector store, they can steer our agent without ever talking to it.
  Here is the concrete version. White text on a white background inside a
  vendor quotation PDF.

READ THE QUOTE OUT LOUD, slowly.

SAY (continue):
  Nobody sees that. It goes through our extraction, it gets embedded, and six
  months later it is the top hit for a pricing question.
  There is no prompt you can write that reliably defends against this. The
  defence has to be architectural: clean it at ingestion, isolate it in
  storage, constrain it at retrieval, and never let text like that be the
  reason a write happens.
  And the sibling problem is memory poisoning - the agent believes something a
  user told it, distils it into long-term memory, and now it is durable fact.
  Time-to-live and a gated distillation step answer that one.


--------------------------------------------------------------------------------
SLIDE 22  PRE-EMBEDDING SANITIZATION - TEN STEPS
--------------------------------------------------------------------------------
SAY:
  So, ingestion. Document lands in a bucket, Eventarc fires, a Cloud Run job
  runs ten steps.
  Validate the file. Extract the text with Document AI. Then step three, which
  is the one everybody skips - I will do it on its own slide. Then
  de-identify with data-loss-prevention. Chunk it. Deduplicate by hash.
  Classify it. Embed it. Upsert it. Record where it came from.
  Two things about this pipeline. If any step fails, the document goes to a
  quarantine bucket and a dead-letter queue - it does not get embedded. Fail
  closed, because a bad chunk is permanent and it poisons every future
  retrieval.
  And the metadata block at the bottom - all of that gets set here, at
  ingestion, because you cannot retrofit it later. Two fields I want to
  highlight: trust level, which is what lets us refuse to act on
  vendor-supplied text later. And embedding model version, so when you upgrade
  your embedding model you know exactly what is stale. Never mix two embedding
  models in one index.


--------------------------------------------------------------------------------
SLIDE 23  STEP 3 - DE-INSTRUCTION
--------------------------------------------------------------------------------
SAY:
  Step three in detail. Six categories of thing to strip.
  Hidden text - white on white, zero opacity, off the page, font size zero.
  You get this from the PDF render tree, not from the extracted string.
  Invisible Unicode - zero-width characters, right-to-left overrides, and the
  Unicode tags block, which is genuinely invisible and genuinely carries text.
  Homoglyphs, where a Cyrillic letter is standing in for a Latin one - fixed by
  Unicode normalisation.
  Active markup, embedded objects in PDFs, and then imperative-sounding
  content, which you catch with a mix of pattern matching and a cheap
  classifier.


--------------------------------------------------------------------------------
SLIDE 24  DROP, OR WRAP AND CONSTRAIN
--------------------------------------------------------------------------------
SAY:
  Once you have found something suspicious you have two choices.
  Drop it, or keep it but wrap it - tag it as untrusted, name the source, and
  attach the injection score.
  Now I want to be honest about that wrapper. It is a hint to the model. It is
  cheap and it is worth doing, but it is probabilistic. The actual control is
  the line underneath: content marked as vendor-supplied or user-supplied can
  never be the only justification for a write. And that is a code check in the
  before-tool hook, not a sentence in the system prompt.
  Choose by corpus. Drop for high-trust things like internal procedures. Wrap
  for corpora that are external by nature - vendor terms, portal responses -
  where you need the content anyway.
  And the sanitiser is one shared library. The same code runs at ingestion and
  at request time. One implementation, two places.


--------------------------------------------------------------------------------
SLIDE 25  STEP 4 - DLP MODES     (cuttable)
--------------------------------------------------------------------------------
SAY:
  Quick but important. De-identification is not one thing, it is three, and
  picking the wrong one breaks you.
  Redaction, for free text with no analytic value. A cryptographic hash, when
  you need to link the same vendor across documents without storing their
  name. And format-preserving encryption, for identifiers you still need to
  validate or join on - PO numbers, vendor IDs. That last one is reversible if
  you hold the key, and it keeps the shape of the original.
  Wrap the key in Cloud KMS. That is what makes crypto-shredding possible
  later, which I will come back to.


--------------------------------------------------------------------------------
SLIDE 26  [*]  VECTOR SEGMENTATION - THREE AXES
--------------------------------------------------------------------------------
SAY:
  One flat index over all company knowledge is wrong three different ways.
  It is a security problem, because one business unit's contract pricing
  becomes retrievable by another. It is a quality problem, because procedure
  text and vendor terms compete and dilute each other. And it is an operations
  problem, because you cannot re-index or expire one corpus without touching
  everything.
  So segment on three separate axes. Who owns it. How sensitive it is. And
  what kind of knowledge it is.
  These are independent, and they compose.


--------------------------------------------------------------------------------
SLIDE 27  THREE SEGMENTATION STRATEGIES
--------------------------------------------------------------------------------
SAY:
  Three ways to do it and they are complementary, not alternatives.
  Option A, one table with tenant and sensitivity columns you filter on.
  Simple and cheap. The risk is in bold - one forgotten WHERE clause is a
  breach.
  Option B, physically separate storage. Strongest guarantee, and it is also
  the right answer if you have data residency rules. It costs more, so use it
  for the top sensitivity tier only.
  Option C, one collection per knowledge type, with a router that picks which
  corpora to search based on the question and on what the caller is allowed to
  see. That improves accuracy and enforces least privilege at the same time,
  which is a nice place to be.
  My recommendation: A plus RLS as the default, B for the restricted tier, C
  for quality.


--------------------------------------------------------------------------------
SLIDE 28  RLS IS THE REAL CONTROL   << DO NOT CUT >>
--------------------------------------------------------------------------------
SAY:
  This is the slide I would defend hardest.
  Do not rely on the application remembering to filter. Turn on row-level
  security in Postgres and let the database refuse.
  Three lines at the top. Enable it. Force it, so it applies even to the table
  owner. Then a policy that says: you can only see rows for your tenant, in
  your business units, at or below your clearance.
  Then at query time we set those three values from the verified identity -
  from the authenticated user, never from anything the model produced - and we
  run the vector search inside that transaction.
  Here is the payoff, in the box. If a developer forgets the filter, the query
  returns zero rows. Not somebody else's data. Zero rows.
  And layer on top of that: each agent gets its own database role, so the
  frontend agent physically cannot select from restricted corpora at all.
  That is the difference between "we filter carefully" and "it is not
  possible". Only the second one survives an audit.

IF ASKED "why AlloyDB rather than a dedicated vector database?":
  Because retrieval here needs to join against live PO state, and because RLS
  is the isolation mechanism I just described. Colocating the vectors with the
  business data gives me both in one consistency domain. Above roughly fifty
  to a hundred million chunks I would move to a dedicated index.


--------------------------------------------------------------------------------
SLIDE 29  RETRIEVAL-TIME QUALITY CONTROLS     (cuttable)
--------------------------------------------------------------------------------
SAY:
  Segmentation makes it safe. This slide makes it accurate.
  Biggest single win in enterprise retrieval: hybrid search. PO numbers and
  part numbers are lexical, and embeddings are genuinely bad at exact
  identifiers. So run the vector search and a keyword search in parallel and
  fuse the two ranked lists.
  Then two-stage retrieval - pull fifty candidates cheaply, rerank them with a
  proper cross-encoder, keep five. That is both cheaper and more accurate than
  putting fifty chunks in the prompt.
  Filter on effective dates, so you never retrieve a superseded price list.
  Cap chunks per document so one verbose PDF cannot dominate.
  And if nothing clears the score threshold, refuse in code. Do not call the
  model with an empty evidence set and hope it says "I don't know".


--------------------------------------------------------------------------------
SLIDE 30  [*]  MEMORY TTL - FOUR TIERS
--------------------------------------------------------------------------------
SAY:
  Memory is not one thing. It is four things with genuinely different physics.
  Working memory - this turn's scratchpad. Minutes, in Redis, expires by
  itself.
  Session memory - the conversation. Weeks, in AlloyDB.
  Long-term facts - distilled durable knowledge. Months to a year, and renewed
  when actually used.
  And then the fourth row, in bold, which is the line I want to hold.


--------------------------------------------------------------------------------
SLIDE 31  AUDIT RECORDS ARE NOT MEMORY
--------------------------------------------------------------------------------
SAY:
  Audit records are not memory. Conflating those two is how teams end up
  either deleting their evidence or retaining personal data forever.
  Memory exists to help the agent do its job. It is reachable by retrieval, it
  expires, and it is deletable on request.
  Audit exists to prove what happened. It is never reachable by retrieval, it
  is immutable for seven years, and the agent's service account cannot read it
  at all.
  Different bucket, different table, different permissions, different
  retention. Not a flag on the same row.
  And for the case where the law says keep it and someone asks you to delete
  it - crypto-shredding. Per-tenant key in KMS. Destroy the key and the data
  is unreadable everywhere, including in every backup. Document the legal
  basis for keeping the ciphertext.


--------------------------------------------------------------------------------
SLIDE 32  DISTILLATION AND THE PROMOTION GATE
--------------------------------------------------------------------------------
SAY:
  Raw transcripts make bad long-term memory. Verbose, expensive to search,
  noisy, and easy to poison. So a nightly job distils them into facts.
  But look at the box in the middle, because that is the security control. A
  candidate fact only gets into long-term memory if at least one of three
  things is true. It came from a system of record, not from someone's prose. A
  human explicitly confirmed it. Or we have independently seen it several
  times, from different users, in different sessions.
  And regardless of all that - we never promote vendor-supplied or
  user-supplied content into a fact that can influence a write.
  Two more things worth saying. Facts are advisory, policy is authoritative -
  if they disagree, policy wins, in code. And we never update a fact in place;
  we insert the new one and close off the old one, so we keep "what we
  believed in August" as well as "what is true now". Audits need both.
  Last line - renewal on use. A fact that gets used gets its life extended. A
  fact nobody has retrieved in ninety days expires. Memory decays by
  usefulness, not just by age.


--------------------------------------------------------------------------------
SLIDE 33  RIGHT TO ERASURE     (cuttable)
--------------------------------------------------------------------------------
SAY:
  Short slide, but it is the question that separates a designed system from a
  demo.
  Someone says: delete everything about this vendor. That one request has to
  reach database rows, vectors, Redis keys, BigQuery partitions, Cloud Storage
  objects, the vector index, and Datadog - who are a third party.
  You cannot do that with a grep. So we write a subject index at ingestion
  time that records, for every subject, which store and which locator. Then
  deletion is a driven, verifiable fan-out that produces a completion
  certificate.
  Design this on day one. Retrofitting it is genuinely painful.



================================================================================
  PILLAR 3 - AUTONOMY AND HUMAN IN THE LOOP
================================================================================

--------------------------------------------------------------------------------
SLIDE 34  DIVIDER - PILLAR 3
--------------------------------------------------------------------------------
SAY:
  Pillar three, and this is the longest section, because this is the gap that
  was completely missing from the original diagram.
  The goal: the agent is fast where that is safe, it provably stops where it is
  not, and a human decision is cryptographically tied to the exact action
  taken.


--------------------------------------------------------------------------------
SLIDE 35  [*]  START WITH AN AUTONOMY LADDER
--------------------------------------------------------------------------------
SAY:
  Before any technology, this table. Five levels, and every tool in the system
  gets assigned to one.
  Level zero, read only. Level one, draft it and let the human do it. Level
  two, prepare the exact payload and execute only after a signed approval.
  Level three, do it and tell them afterwards. Level four, just do it.
  I have mapped the real tool names from the diagram into the right-hand
  column, and level two is the row in bold: releasing an order, rejecting an
  order, and high-value vendor submissions.
  Everything else in this pillar is machinery to implement this one table. And
  the important thing is that this table is a business decision, not an
  engineering one - Procurement, Finance and Risk sign it, not me.


--------------------------------------------------------------------------------
SLIDE 36  TWO RULES THAT FOLLOW
--------------------------------------------------------------------------------
SAY:
  Two consequences, and both are slightly counterintuitive.
  Rule one. The unattended scheduled path gets a lower ceiling, not a higher
  one. There is no human present, so the batch job may prepare an action and
  park it for approval, but it may never approve its own work. People's
  instinct is the opposite - "it is a trusted internal job, let it run" - and
  that instinct is how you get a bad night.
  Rule two, and this is the one that shapes the code. The autonomy level is a
  property of the tool AND its parameters. Releasing five hundred dollars
  might be level three. Five hundred thousand is level two with two approvers.
  Any amount with a bank detail change is level two plus an out-of-band phone
  call.
  Which means autonomy is a function evaluated in code on every single call.
  Not a static label. And definitely not something the model negotiates.


--------------------------------------------------------------------------------
SLIDE 37  DETERMINISTIC BREAKPOINTS - FOUR REQUIREMENTS
--------------------------------------------------------------------------------
SAY:
  A breakpoint is a mandatory pause before an action with a side effect. The
  word deterministic is doing four jobs.
  One, it is evaluated outside the model. It is never "the model decides to
  ask permission". Prompts are probabilistic, and a two million dollar release
  is not a place for probability.
  Two, same inputs give the same decision every time.
  Three, it is versioned and unit tested, because policy is code.
  Four, it is unbypassable - it sits at the boundary the model has to pass
  through.
  Then three non-negotiables at the bottom. Separation of duties, enforced by
  a database constraint, not application logic. Fail closed - if the policy
  engine is unreachable, we pause; an error must never be more permissive than
  a rule. And show the human what will actually happen: the resolved payload
  and the difference against current SAP state, not a summary the model wrote.
  The human approves the payload. The model does not get to describe it.


--------------------------------------------------------------------------------
SLIDE 38  POLICY AS DATA AND CODE - THE TOOL MANIFEST
--------------------------------------------------------------------------------
SAY:
  Two artefacts, and separating them matters.
  This one is the tool manifest. For each tool: does it write, what is the
  system of record, can it be undone, how big is the blast radius, is it safe
  to retry, and what is its default level.
  These are physics. They only change when the tool changes.
  Read the box. Keeping this separate from the thresholds means a threshold
  change is a reviewed data change with its own version number - not a code
  deploy. Finance can tune limits without an engineer.
  And the last line is a nice property. Every tool the agent can call must
  have a manifest entry. No entry means the policy engine falls through to
  deny. Registration is not optional.


--------------------------------------------------------------------------------
SLIDE 39  POLICY AS DATA AND CODE - THE APPROVAL POLICY
--------------------------------------------------------------------------------
SAY:
  And this is the part that changes often. Version seven, as an example.
  Over fifty thousand dollars, a manager approves. Over five hundred thousand,
  two people approve. Price varies more than five percent, category manager.
  New or risky vendor, vendor management. Any rejection needs a human, because
  you cannot un-reject a supplier.
  Bank detail change - two approvals and an out-of-band confirmation. That is
  the classic invoice-fraud vector and it deserves its own rule.
  Then the unattended ceiling I mentioned. And low confidence - if retrieval
  was weak or the evidence was vendor-supplied, a human decides.
  And the last line. Default deny. Anything I did not think of pauses rather
  than proceeds.

POINT AT: the default line.


--------------------------------------------------------------------------------
SLIDE 40  WHERE IT EXECUTES   << DO NOT CUT >>
--------------------------------------------------------------------------------
SAY:
  This is the whole enforcement mechanism, and it is about twenty lines.
  In the before-tool callback we build a context from the verified identity,
  evaluate the policy, and always record the evaluation - whether we allowed
  it or not.
  If it is allowed, return nothing and the real tool runs.
  If it needs approval, we write a pending approval row - and note what goes
  into it. The arguments. A hash of those arguments. The exact thing the
  approver will see. Who asked, so we can check separation of duties later.
  An expiry. And a serialised snapshot of the run so we can resume it.
  Then we publish an event and return a status object.
  Now read the box, because this is the point. The tool never runs. The model
  gets back a status it can relay to the user - "submitted for approval" - but
  there is no code path from the model to the side effect. The pause is
  structural. It is not the model being persuaded to behave.


--------------------------------------------------------------------------------
SLIDE 41  [*]  ASYNCHRONOUS ORCHESTRATION - WHY MANDATORY
--------------------------------------------------------------------------------
SAY:
  A human approval takes minutes to days. So you cannot hold an HTTP
  connection, you cannot hold a Cloud Run instance, and you cannot hold a
  Python object in memory.
  Which means the run has to become durable, external and resumable. Three
  moves. Checkpoint everything to the database. Release all compute - return
  a two-oh-two and a status URL, let the instance exit, pay nothing while
  waiting. Then rebuild from the checkpoint when the approval arrives,
  including restoring the trace context.
  This is the clearest single line between a demo and a production agent.
  And it needs three services doing three different jobs. Do not make one
  service do all of it.


--------------------------------------------------------------------------------
SLIDE 42  THE END-TO-END HITL FLOW
--------------------------------------------------------------------------------
SAY:
  Walk this left to right and top to bottom. It is the most important diagram
  in the deck.
  Breakpoint hits. Two things happen in parallel - we checkpoint to AlloyDB,
  and we publish an event. The event fans out to a notification and to Cloud
  Workflows, which registers a callback and starts a timeout clock. Workflows
  can wait up to a year, which is why it is the right tool.
  Meanwhile our Cloud Run instance exits. Zero cost while waiting.
  The approver gets a link, authenticates properly, sees the rendered payload
  and the difference against SAP, and decides. That decision gets signed by
  Cloud KMS.
  Then the resumption handler does seven checks - I have a whole slide on
  those. Then, and only then, Cloud Tasks executes the tool with an
  idempotency key, we append to the audit trail, and we tell the original user.
  And the run ID and trace ID travel the entire path, including through the
  message attributes on Pub/Sub. That is pillar one paying for itself.


--------------------------------------------------------------------------------
SLIDE 43  CHOOSING THE ORCHESTRATOR     (cuttable)
--------------------------------------------------------------------------------
SAY:
  Briefly, why four services and not one.
  Cloud Workflows owns the waiting, the timeout and the escalation, because
  await-callback is purpose-built for exactly this.
  Cloud Tasks does the throttled execution afterwards, because it gives you
  rate limiting and deduplication per queue.
  Pub/Sub carries the events, because more than one thing cares.
  And AlloyDB is the source of truth. If Workflows and the database disagree,
  the database wins.


--------------------------------------------------------------------------------
SLIDE 44  THREE HARD PROBLEMS
--------------------------------------------------------------------------------
SAY:
  Three problems that are each a production incident waiting to happen.
  One, idempotency. Pub/Sub redelivers. Cloud Tasks retries. Humans
  double-click. Without protection you release the same purchase order twice.
  So we derive a key from the run, the step, the tool and a canonical form of
  the arguments, and we insert it with a unique constraint BEFORE calling SAP.
  If the insert conflicts, we return the stored result instead of calling
  again. The database makes duplication impossible.
  Two, timeouts. A scheduled sweep every fifteen minutes: reminder at four
  hours, escalate at twenty-four, expire at seventy-two and release any lock.
  Every state has an exit. There is no "waiting forever" state.
  Three, races - and the important one is at the bottom. The approval
  authorises an intent. By the time we execute, SAP state may have moved: the
  order changed, the price moved, the vendor got blocked. So we re-validate
  the preconditions at execution time and abort if they drifted.


--------------------------------------------------------------------------------
SLIDE 45  RUN STATE MACHINE AND SAGA
--------------------------------------------------------------------------------
SAY:
  Top half, the state machine. Nothing exotic - the point is just that every
  state including every failure has a defined exit.
  Bottom half is the uncomfortable bit. There is no distributed transaction
  across SAP and a third-party vendor portal. So a two-step action can half
  succeed: SAP accepts the release, the vendor portal returns a five-oh-three.
  Now two systems disagree and retrying does not fix it.
  So every step gets a defined compensating action - an SAP reversal document,
  a cancellation to the portal - and we drive a compensating state explicitly.
  And if compensation itself fails, we escalate to a human with the full trace
  and close it as an exception. We never silently give up, and we never leave
  two systems inconsistent without a record.
  This, by the way, is why pillar one comes first. You cannot compensate if
  you do not know precisely what succeeded and in what order.


--------------------------------------------------------------------------------
SLIDE 46  [*]  CRYPTOGRAPHIC RESUMPTION   << DO NOT CUT >>
--------------------------------------------------------------------------------
SAY:
  The resumption path is the highest-value target in the whole system. If I
  can forge a resumption, I can make your agent do anything.
  So five properties, and each one kills a specific attack. Read across.
  Integrity, from a signature - stops forging. Payload binding, from a hash of
  the arguments re-checked at execution - stops approving one thousand and
  executing one million. Single use, from burning an identifier - stops
  replaying one approval ten times. Expiry - stops a stale approval being used
  weeks later. And non-repudiation - stops "I never approved that".
  Notice these are five different attacks. A signature alone gets you one of
  the five.


--------------------------------------------------------------------------------
SLIDE 47  THE APPROVAL TOKEN
--------------------------------------------------------------------------------
SAY:
  The token itself. Mostly standard claims - who, when, expires, single-use
  identifier. Plus our run ID and step so we know what to resume.
  Two fields carry unusual weight and they have the arrows.
  Args-hash is a hash of the exact payload to be executed, and we recompute
  and compare it at resumption. If anything changed between approval and
  execution, the token is invalid. That is what makes "approve small, execute
  large" impossible.
  UI-render-hash is a hash of what was actually drawn on the approver's
  screen. That is the "what you see is what you sign" guarantee. It proves
  they saw this vendor, these line items, this total. Without it, a
  compromised approval screen could show one thing and sign another and you
  could never prove which.
  And the key ID in the header means rotation does not break old tokens.


--------------------------------------------------------------------------------
SLIDE 48  SIGNING WITH KMS - WHY ASYMMETRIC     (cuttable)
--------------------------------------------------------------------------------
SAY:
  Left side is configuration - hardware-backed key, elliptic curve, ninety-day
  rotation, and the private key never leaves KMS. Which gives us a bonus:
  Google logs every signing operation in the audit log, outside our
  application's control. So there is an independent record of every approval
  ever signed. If our table and that log disagree, we have detected tampering.
  Right side is the design decision. Why not a shared secret?
  Because the service that executes must be able to verify, but must not be
  able to create approvals. With a shared secret, compromising the executor
  lets an attacker approve their own work. Asymmetric signing splits those two
  powers, and that split is the entire security property.


--------------------------------------------------------------------------------
SLIDE 49  VERIFICATION - THE ORDERED CHECKLIST
--------------------------------------------------------------------------------
SAY:
  Twelve checks, in order, fail closed at every one.
  I will not read them all. I want you to look at five, seven and eleven,
  because those are the three that get skipped in real implementations, and
  each one is a complete exploit on its own.
  Skip five, and one approval releases the order ten times. Skip seven, and
  you approve a thousand dollars and execute a million. Skip eleven, and you
  execute against state that changed after the human looked at it.
  Then the bottom section, which is a distinction people collapse. The token
  authorises the decision. It does not authenticate the person. Email links
  get forwarded and inboxes get compromised. So the approval screen sits behind
  proper authentication with multi-factor, and we bind the token's subject to
  that authenticated identity at signing time. The email is only a pointer to
  the screen.


--------------------------------------------------------------------------------
SLIDE 50  TAMPER-EVIDENT AUDIT
--------------------------------------------------------------------------------
SAY:
  Last slide of this pillar. A compliance reviewer needs to be able to
  disprove tampering, not take our word for it.
  So three independent copies. One, a hash chain in the database - each row
  includes the hash of the previous row, so any retroactive edit breaks the
  chain and a nightly job catches it. Two, we anchor the chain head to a
  write-once object in Cloud Storage with a retention lock, which even a
  project owner cannot delete. Three, Google's own KMS audit log, which
  records every signature independently of us.
  Read the box. An insider who can write to our database still cannot rewrite
  history, because they would also need to break a locked storage object and
  Google's audit log.



================================================================================
  PILLAR 4 - INPUT MISUSE TESTING AND RED TEAM
================================================================================

--------------------------------------------------------------------------------
SLIDE 51  DIVIDER - PILLAR 4
--------------------------------------------------------------------------------
SAY:
  Last pillar. Every input and output passes a mandatory checkpoint, and
  nothing ships without automatically proving the defences still hold.


--------------------------------------------------------------------------------
SLIDE 52  [*]  THREAT MODEL FIRST
--------------------------------------------------------------------------------
SAY:
  Threat model before controls, and grounded in named frameworks so this is
  not just my opinion - OWASP's list for language model applications, MITRE
  ATLAS, the NIST AI risk framework, and Google's own SAIF.
  Nine threats, each made concrete for this system. I will pull out three.
  Indirect prompt injection - hidden instructions in a vendor PDF. That is the
  bolded row and it is the one I worry about most.
  Excessive agency - "approve all pending orders".
  And the confused deputy at the bottom - a vendor response engineered to
  trigger one of our privileged tools. Which is why tool responses get treated
  as untrusted input and re-screened.


--------------------------------------------------------------------------------
SLIDE 53  [*]  THE LOAD-BEARING POINT   << DO NOT CUT >>
--------------------------------------------------------------------------------
SAY:
  I want to say this plainly, because any design that claims otherwise is the
  one you should worry about.
  Prompt injection is not fully solvable. So we design the system such that a
  successful injection is not sufficient to cause harm.
  A completely jailbroken PO agent still cannot release a two million dollar
  order. The breakpoint is in code. Execution needs a signed, single-use token
  bound to the payload hash. Row-level security stops it reading another
  business unit's data. The unique constraint stops it acting twice.
  Guardrails reduce probability. Architecture caps impact. Lead with the
  architecture.
  And the third bullet is what makes this an engineering claim instead of a
  slogan: we test it. The red team runs against a mocked SAP that fails the
  test if an unapproved write ever arrives.


--------------------------------------------------------------------------------
SLIDE 54  I/O GUARDRAIL MIDDLEWARE
--------------------------------------------------------------------------------
SAY:
  Two mandatory chokepoints - everything in, everything out.
  On the way in: web application firewall and rate limiting at the edge, then
  authentication, then in the application - schema and length caps, Unicode
  normalisation, Model Armor for injection and jailbreak screening,
  data-loss-prevention to stop secrets going in, and an intent check.
  On the way out: schema conformance, a groundedness check that every claim
  cites something we actually retrieved, an egress scan, URL allow-listing so
  a markdown image cannot smuggle data out, and business invariants.
  Two design notes at the bottom. First, this is middleware - one shared
  library imported by both agents and installed as callbacks. Coverage becomes
  structural instead of depending on a developer remembering.
  Second, and this is the step most teams miss: we re-run the input chain on
  tool responses. Vendor portal HTML and SAP free text are inputs too, and
  they are the likeliest injection vector. Guarding the user boundary and
  leaving the tool boundary open defeats the whole pillar.


--------------------------------------------------------------------------------
SLIDE 55  BUSINESS INVARIANTS - THE SANITY FIREWALL
--------------------------------------------------------------------------------
SAY:
  My favourite slide, because there is no AI on it at all.
  Before any write, a list of deterministic cross-checks. Do the line items
  add up to the header total. Does the vendor's bank account match the vendor
  master in SAP - that single check is probably the highest-value control in
  the entire system. Is the currency right for this contract. Is the quantity
  within contract limits. Is the price within tolerance. Has this order
  already been released - checked in SAP, not from memory.
  Right-hand side is why this layer matters most. It catches hallucination and
  successful injection with the same code, because a hallucinated total and an
  injected total both fail "lines must sum to total". It is deterministic. It
  is microseconds. A finance reviewer can read the list and agree with it. And
  every violation we find in production becomes a permanent new rule.
  Failure means block and park for a human. Never warn and proceed.


--------------------------------------------------------------------------------
SLIDE 56  FAIL OPEN VERSUS FAIL CLOSED
--------------------------------------------------------------------------------
SAY:
  Short but it is the question a good reviewer will ask: what happens when
  Model Armor times out.
  If you do not have a documented answer, the answer is wrong.
  Ours is split by path. On the write path, fail closed - block, park, alert.
  An unscreened write is unbounded risk and availability is not worth a wrong
  purchase order.
  On the read path, degrade instead - serve the request, mark the run as
  unscreened, alert, and disable every write tool for that run. Buyers still
  get answers, and the risk is bounded precisely because writes are already
  off.
  Ingestion fails closed, because a bad chunk is permanent.


--------------------------------------------------------------------------------
SLIDE 57  [*]  AUTOMATED EVALUATION GATES
--------------------------------------------------------------------------------
SAY:
  Prompts are code. A one-word change to a system prompt can silently break
  tool selection or reopen a jailbreak, and no unit test catches it.
  So we need a blocking, automated, quantitative check in the pipeline.
  Nothing reaches production without passing.
  And the agent-specific piece is trajectory evaluation, which matters most.
  A correct final answer, reached by accidentally calling the release tool
  twice, is a failure - even though the text looks fine. Agent quality lives
  in the sequence of actions, not the prose. ADK's evaluation framework scores
  exactly that.
  Two governance details. Staging uses mocked SAP, so an evaluation run can
  never post a real order. And thresholds are compared against the currently
  deployed version, not against fixed numbers - that is what catches slow
  degradation.
  And one more, which is the classic failure of this control: changing the
  evaluation set needs the same review as changing the code. Otherwise someone
  lowers the bar to get a release out.


--------------------------------------------------------------------------------
SLIDE 58  THE CI/CD GATE
--------------------------------------------------------------------------------
SAY:
  Concretely. Push triggers Cloud Build. Lint and unit tests. Then policy
  tests, which must be one hundred percent. Build and scan the container.
  Deploy to staging with everything mocked. Run the trajectory evaluation. Run
  the autorater metrics. Run a fast subset of the red team - zero tolerance
  there. Check cost and latency against the deployed version. Then attest the
  image.
  Then Cloud Deploy rolls out to five percent of traffic, then fifty, then a
  hundred, with automatic rollback on error rate or a spike in guardrail trips.
  Bottom section is the datasets. The one I want to point at is regression -
  every production incident becomes a permanent test. That list only grows.


--------------------------------------------------------------------------------
SLIDE 59  GATE THRESHOLDS     (cuttable)
--------------------------------------------------------------------------------
SAY:
  Illustrative numbers - you tune these with real traffic. Two rows matter.
  Critical attack success rate must be zero, hard block, no override. Any
  bypass of a breakpoint is a release blocker, full stop.
  And false refusal rate at or below five percent, which brings me to the
  point I will make again in a minute - a guardrail that blocks real buyers
  gets switched off by the business, and that is a worse outcome than a
  slightly leakier one.


--------------------------------------------------------------------------------
SLIDE 60  RED-TEAM HARNESS
--------------------------------------------------------------------------------
SAY:
  Hand-written attack prompts go stale in weeks. So this is a loop, not a
  checklist.
  Seed corpus of attack families. An attacker model generates variants -
  paraphrase, translate, encode, role-play, multi-turn, and hide it in a PDF.
  Those run against a staging copy with the full stack switched on. A judge
  decides what happened. Results go to BigQuery and we trend attack success
  rate by family over time. Any success becomes a priority ticket, a permanent
  regression test, and a patch.
  Four design points at the bottom, and I will give you two.
  Deterministic assertions beat model judging wherever possible. "Did the
  release tool get called" is answerable from the mock's call log. No judge, no
  judge error. Save the model judge for genuinely fuzzy questions.
  And attack success means harm, not rudeness. If no forbidden action happened
  and nothing leaked, an impolite refusal is not a breach. Measuring "did it
  say something bad" optimises the wrong thing.


--------------------------------------------------------------------------------
SLIDE 61  [*]  QUALITY METRICS - SIX LAYERS
--------------------------------------------------------------------------------
SAY:
  Six layers, top to bottom, and the top one is the one your sponsor cares
  about.
  Business outcome - straight-through processing rate, cycle time, how much
  human touching is still needed, dollars processed autonomously.
  Then agent quality, safety, human-in-the-loop health, reliability, and cost.
  And the line at the bottom is the north star: cost per successfully and
  correctly processed purchase order, with zero unauthorised actions. Both
  halves of that sentence matter. Cheap and wrong is not a win.


--------------------------------------------------------------------------------
SLIDE 62  TWO METRICS THAT CARRY UNIQUE INFORMATION
--------------------------------------------------------------------------------
SAY:
  Two metrics I would put on the wall.
  First, override and reversal rate. If humans reverse thirty percent of the
  agent's proposals, the agent is not helping. But if they approve
  ninety-nine point nine percent within four seconds, they have stopped
  reading and your approval control is now theatre. Both extremes are alarms.
  Healthy is somewhere around two to ten percent. And genuinely - alert when an
  approver's median decision time drops below a plausible reading time.
  Second, false refusal rate. The easiest way to score perfectly on safety is
  to refuse everything. So always report block rate and false positive rate as
  a pair. And run new guardrails in shadow mode first - log what they would
  have blocked, measure it on real traffic, then switch to enforcing.



================================================================================
  SYNTHESIS
================================================================================

--------------------------------------------------------------------------------
SLIDE 63  DIVIDER - SYNTHESIS
--------------------------------------------------------------------------------
SAY:
  Right. Let me put it all in one picture and then talk about delivery.


--------------------------------------------------------------------------------
SLIDE 64  [*]  TARGET ARCHITECTURE
--------------------------------------------------------------------------------
SAY:
  This is everything from the four pillars in one diagram. I am not going to
  read it. Let me trace one path through it and you will have the shape.
  A buyer arrives top left. Load balancer, firewall, authentication, then the
  frontend agent. It calls the backend orchestrator with a signed identity
  token. The orchestrator plans, screens input through Model Armor, retrieves
  from AlloyDB under row-level security, and hits a breakpoint. It checkpoints
  to AlloyDB and publishes an event. Workflows starts waiting. Instance exits.
  Approver comes in top right, through authentication, reviews the payload,
  signs with KMS. The decision comes back through Pub/Sub, gets verified,
  Cloud Tasks executes it out through the network egress path bottom right to
  SAP. Every step emits telemetry to the collector sidecar, which fans out to
  Cloud Trace and Datadog.
  And along the bottom: the ingestion pipeline on the left, CI/CD, and the
  cross-cutting controls - service perimeter, customer-managed keys, secret
  management.

IF ASKED "is this over-engineered?":
  For a chatbot, yes, enormously. For something that releases purchase orders
  into SAP, no - and phase zero to two of my plan is only about five weeks. I
  will show you the sequencing in a moment.


--------------------------------------------------------------------------------
SLIDE 65  [*]  GCP SERVICE INVENTORY
--------------------------------------------------------------------------------
SAY:
  Same thing as a table, with dots showing which pillar each service serves.
  About thirteen rows and twenty-five services.
  The thing to notice is how much of it is boring. Cloud Run, Postgres, Redis,
  object storage, a warehouse, a queue. The only genuinely AI-specific rows
  are Vertex AI and Model Armor. Most of governing an agent is ordinary
  distributed systems engineering done properly.


--------------------------------------------------------------------------------
SLIDE 66  [*]  THE FIVE INTEGRATION SEAMS
--------------------------------------------------------------------------------
SAY:
  If someone is going to probe this design, they will probe the joins, not the
  boxes. So here are the five joins with the specifics.
  Frontend to backend - private ingress, one narrow permission, an identity
  token scoped to the backend, and the verified user identity in a signed
  internal header. Never a plain header the model could influence.
  Backend to AlloyDB - private networking, IAM database auth, a separate role
  per agent, and those three session variables set from verified identity so
  row-level security does the work. And size your connection pool for Cloud
  Run concurrency - a hundred instances at ten connections each will exhaust
  the database, and that is a real outage people hit.
  Backend to SAP and the vendor portals - egress through a NAT with a reserved
  static IP so vendors can allow-list us, Apigee in front of SAP for quota,
  credentials from Secret Manager, and every response treated as untrusted.
  Fourth is the approval chain end to end, which we walked through.
  Fifth is telemetry fan-out, including redaction before anything leaves for
  Datadog.


--------------------------------------------------------------------------------
SLIDE 67  IMPLEMENTATION ROADMAP
--------------------------------------------------------------------------------
SAY:
  Seven phases, roughly twenty weeks, and each phase has an exit criterion
  rather than a deliverable list, because "done" should be testable.
  Phase zero, foundations and Terraform. Phase one, observability. Phase two,
  cost and loop breakers. Phase three, RAG governance. Phase four, autonomy
  and human-in-the-loop, which is the biggest. Phase five, guardrails and
  evaluation gates. Phase six, hardening and scale.
  Three to five can overlap once phase one lands. Phase one is on the critical
  path for everything, and the next slide says why.


--------------------------------------------------------------------------------
SLIDE 68  WHY THIS SEQUENCE     (cuttable)
--------------------------------------------------------------------------------
SAY:
  One line each.
  Observability first, because you cannot debug or prove anything without
  traces, and every later phase needs them to demonstrate it works.
  Breakers second, because phases three to five involve running the agent
  thousands of times against adversarial input.
  RAG before autonomy, because granting write access over a poisonable
  knowledge base is the worst possible order to do this in.
  Then human-in-the-loop. Then gates, which lock in everything above so it
  cannot regress.
  And the last bullet is the anti-pattern - build the agent, then add
  governance at the end. By then the tool boundary is not a single chokepoint
  any more, and almost none of this is retrofittable.


--------------------------------------------------------------------------------
SLIDE 69  KEY TRADE-OFFS
--------------------------------------------------------------------------------
SAY:
  Seven decisions where a reasonable person could disagree, with my
  recommendation and the reason.
  AlloyDB rather than a dedicated vector database, because retrieval needs to
  join to live PO state and because row-level security is my isolation
  mechanism.
  Cloud Run rather than the managed agent runtime, because I want the sidecar
  and the network control.
  KMS asymmetric rather than a shared secret, because the executor must verify
  without being able to mint.
  And both Datadog and Cloud Trace, from one collector, because that costs
  almost nothing extra and Datadog was already in the diagram.
  If you want to challenge one of these, challenge the first.


--------------------------------------------------------------------------------
SLIDE 70  TOP RISKS
--------------------------------------------------------------------------------
SAY:
  Ten risks with likelihood and impact. I will name the one I actually lose
  sleep over, and it is row two, not row one.
  Approval fatigue. Humans rubber-stamping. Because it is high likelihood and
  high impact, and unlike the technical risks it degrades quietly. Nobody
  files a ticket saying "I have stopped reading the approvals."
  Mitigations are behavioural, not technical - tune the thresholds so
  approvals are rare enough to feel meaningful, show only the difference
  rather than the whole payload, monitor time-to-decision, and require a typed
  justification above a certain value.
  Row six is the other underrated one - a model upgrade silently degrading
  tool selection. That is what the trajectory gate is for.


--------------------------------------------------------------------------------
SLIDE 71  GO-LIVE GATES
--------------------------------------------------------------------------------
SAY:
  Nine gates, all green, because "we think it is fine" is not a launch
  decision.
  Every write tool mapped to a level and signed off by the business. Zero write
  paths bypassing the policy engine, verified by review and by an automated
  assertion. Approval tokens tested negatively - we prove the replay fails and
  the tampered payload fails. Critical attack success rate at zero on the
  latest nightly run. Cost per order within budget and the kill switch tested
  end to end. Audit chain verified. Runbooks written. Rollback rehearsed. And
  the data protection review done, including demonstrating a deletion request
  end to end.


--------------------------------------------------------------------------------
SLIDE 72  [*]  FIVE THINGS TO TAKE AWAY
--------------------------------------------------------------------------------
SAY:
  Five things, and then I will stop.
  One. The model plans, code enforces. Budgets, approvals, isolation and
  exactly-once are all enforced by mechanisms that cannot change their mind.
  Two. Two identifiers. One for debugging that expires, one for audit that
  does not.
  Three. Retrieved content is prompt content. Clean it before embedding,
  isolate it in the database rather than in application code, and never let
  low-trust text be the only reason a write happens.
  Four. A human decision is bound to one exact payload - signed, single-use,
  time-limited, re-verified at execution, and re-checked against current SAP
  state.
  Five. Prompt injection is not solvable, so we made it insufficient. And then
  we proved it, on every deploy, against a mock that fails the build if an
  unapproved write ever arrives.
  That is the design. Happy to go deeper on any of it.



================================================================================
  Q&A PREPARATION
================================================================================

Q: This looks like a lot. What is the minimum viable version?
A: Phases zero to two - about five weeks - plus the autonomy ladder and
   breakpoints from phase four. That gives you traceability, cost safety, and
   no unapproved writes. Cryptographic resumption and the full red-team harness
   can follow. But I would not go live writing to SAP without the breakpoints.

Q: Why not use Vertex AI Agent Engine and get sessions and tracing managed?
A: It is a real option and it would save work. I chose Cloud Run because I want
   the collector sidecar and direct VPC egress with a static IP for vendor
   allow-listing. If those two constraints go away, Agent Engine gets more
   attractive.

Q: Isn't a human in the loop going to destroy the throughput benefit?
A: Only if you put humans in the wrong place. Six of the ten tools are
   read-only and stay fully autonomous. The approvals cluster on high value,
   price variance, new vendors and bank changes. Straight-through processing
   target is seventy percent. And the metric on slide 62 tells you if the
   thresholds are wrong in either direction.

Q: How do you know the guardrails actually work?
A: I do not trust the claim, I test it. The red team runs against a mocked SAP
   that records every call and fails the test if an unapproved write arrives.
   That tests the architecture, not the prompt. And it is a blocking gate on
   every deploy, not an annual exercise.

Q: What if the LLM provider changes the model underneath you?
A: That is risk six. Pin model versions, and gate on trajectory evaluation
   compared against the currently deployed revision rather than an absolute
   number, so a two-point drop blocks the release. Plus canary with automatic
   rollback.

Q: Who owns the approval policy?
A: Not engineering. Procurement, Finance and Risk own the thresholds - that is
   exactly why the policy is versioned data separate from the tool manifest, so
   they can change a limit without a code deploy. Engineering owns the engine
   and the tests.

Q: Is Datadog necessary if you have Cloud Trace?
A: Not strictly, but it was in the given architecture and teams have existing
   dashboards and on-call workflows there. The point is that dual export costs
   one collector configuration, so I did not have to choose. I do want Cloud
   Trace as well, for native log-to-trace correlation and no egress.

Q: What is the single biggest weakness in your design?
A: Approval fatigue, and I would say that honestly rather than name a technical
   one. Every cryptographic control I described assumes the human actually
   looked. That is why the render hash and the decision-time monitoring are in
   there, but it is a mitigation, not a solution.

Q: How much will this cost to run?
A: I did not have volume numbers, so I designed the instrumentation to answer
   it rather than guessing - billing export joined to the run table gives cost
   per order. The controls that keep it down are on slide 18, layer six: cheap
   model for extraction, expensive model only for planning, context caching,
   and reranking instead of stuffing context.

Q: What about the acronyms you could not expand?
A: Marked as assumed in the written notes. The design keys off each tool's side
   effect and blast radius, not its name, so correcting them changes labels and
   nothing structural. But I would want them corrected before this goes to
   anyone at Ingram.


================================================================================
  DELIVERY REMINDERS
================================================================================
  - Slow down on slides 7, 28, 40, 46 and 53. Those are the argument. Pause
    after each one.
  - Read the injected-PDF quote on slide 21 out loud. It lands better spoken.
  - On the big diagrams (4, 42, 64) trace ONE path with your finger. Do not
    narrate the whole picture.
  - Say "I do not know" if you do not know, then say what you would do to find
    out. That reads as senior. Guessing does not.
  - Use they/them for the buyer, the approver and the attacker throughout.
  - Watch the clock at slide 34. If you are past the halfway point in time but
    only at slide 34 of 72, start using the cuttable list.
================================================================================
