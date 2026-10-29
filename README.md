Hi, Blotato!

I think it's valuable to see the thought process, so I share everything:
- the (good enough for a homework) result: [./result](./result),
- the chat itself: https://claude.ai/share/e096d1e1-4c58-4b29-a669-9f50d6224099,
- and the hidden prompts, which Claude refuses to share: ↓.

Thanks!

# Prompt #2

Ok, Claude, so I found a few hours max to iterate on this. Let's organize our work in 2 steps:

First, I'll write my raw thoughts about how I would approach this task. Please roast me: point on weak spots, tell me if I'm wrong, inaccurate or there are better or more modern ways to do anything you feel needs to be improved.

Second, when we settle all the disputes, please create a single README.md which completely describes the resulting approach. It should include sections of architecture, data model, execution model, and important considerations. For every section (optionally except the considerations) create a corresponding D2 diagram, include its raw source as code block and insert its generated image too (if you can't generate D2 images from source, tell me and just insert placeholders, I'll generate from the sources and insert the images into their placeholders myself). Plese, be conscise and straight to a point, the document should be as short as possible without losing important details.

---

Ok, my thoughts.

Given that the task says we can assume the system already has the capability to receive and send both comments and direct messages, it seems like we're asked to create a subsystem alongside to allow user automations. I'll consider as given that we have some kind of contract like this one:
- two events which fire when a comment or a direct message is received,
- two API endpoints which we could call to send a comment or a direct message.

Possible data types might be something like this:
- Dm: fromUserId, id, timestamp (no toUserId here, because we must not allow users to interfere with other user's data, so even though our bigger system might persist dms including both ids, in our subsystem we should always strip the own id to not even introduce a whole set of issues related to messing with other users);
- Comment: fromUserId, id, text, timestamp, toCommentId, toPostId;
- Post: fromUserId, id, text.

So now we need an automation. A user should be able to achieve something like "wait for a comment -> send a comment -> wait for a dm -> send a dm". Quite obviously we cannot run this as a single script because at the the waiting steps of it might delay arbitrarily. If it was an in-house solution, instead of throwing in complexities, I would prioritize to investigate modern approaches like Temporal, which might be a perfect solution, but here we deal with an end user, so we have to buy simplicity for the user even while paying by more complexity on our end. The obvious solution is a state machine. Maybe we could make use of Step Functions - which are basically a state machine - or their alternative, but I'm not into spending excessive time for this given that we're doing it within a take home assignment, not a real business. So let's just describe how we would introduce state machine without diving into vendor specifics.

Basically, the state machine is a directed graph. Nodes seem to be like: wait for a comment by toPostId and wait for a dm. For the sake of observability I would also include the start and end nodes, but it's just a gut feeling for now. Edges should be connections between nodes. I'm thinking about some n8n-like board with nodes, where user adds edges manually instead of from inside an automation script, but it's the whole another discussion, which starts with how a user might configure their automation?

In the most simple way we could hardcode every node type and every edge type and every logic type, like "receive a dm -> script: if text contains {configured-input-text}, proceed with the outgoing edge0, else edge1". This might look simple in a good UI, but it very much lacks flexibility. That's why I'm thinking more about giving a user an ability to write an actual script. In which language? Idk, some popular one like Python or TypeScript, or even a custom one (but this is an overkill not worth discussion for now). Important questions will be how we run this free-type script. First, it can be malware, it can introduce an infinite loop, etc. And second, it has to contain some predefined code, like a set of libraries to deal with our system. To solve the first issue we should use an isolated environment. It might be too expensive to provision for every step, so I would recommend just using Lambdas or their analogs, which are simple and solve these security considerations for free. While using Lambda, we can actually include a predefined script too, like our library.

Actually, it seems to me that giving out that much control might be an overkill too. Maybe we should stick to some predefined script interface instead. Like whatever you write here should return an instance of the next node, like { id, ...args }. Or maybe it's better to deny using dynamic jumps across nodes so that the whole set of next nodes should be first explicitly created via a UI and be connected with output edges for the node to be able to jump into. Maybe we should introduce more node types, like the "if" node with a simple script and multiple output connections or the "all/fanout" node. It starts looking more and more complex while I'm thinking... So maybe let's fallback to a reasonable compromise. Let's introduce a "state" object. So as we have an instance of Process (id, userId), we'll save the current state into the process.state field. Obviously we should limit the size of it to not blow our database. But it makes easy to track what node changed what and it becomes easy to just provide every script with the whole incoming state object and expect its partial changes in a return object. Ok, so we have an instance of Process, inside it we have multiple Steps (nodes), inside every step we have some code, which we run via a Lambda, giving it a process.state as input and expecting its partial updates as output. Every step has a status field (like todo, running, failure, success). Our subsystem basically has either a queue of steps to perform or it constantly scans for pending steps and runs them. Btw, there probably should be a cron step type too. So the subsystem has a flow of events, by which it selects corresponding steps inside not finished processes, adds them to a queue, spawns one-time isolated workers, runs and updates their statuses. So far so good. We should also make sure users don't abuse the system, like enqueueing a 1000 of next steps. Maybe we should introduce a hard limit. Obviously we're not going to allow any asyncronous APIs to be run inside a script, because it will not handle errors properly, it might timeout, it might abuse the system, etc. It means by previous idea about including some library into every script was probably bad, let's not do this. The script is always simple: old process state in, new process state out. Looks good to me. Btw, we can and probably should include the toPostId value into the step which waits for a comment for this toPostId to be able to do a database scan for all waiters if we receive the corresponding comment. Same with userId for waiting for a dm.

What about edges? Well, we can settle into the concept where the process.state contains the next step (or steps) so that the user might just set this field into whatever stepId they see fit. It gives more flexibility, but with probably more complexity for the user. And also the UI becomes just a list of steps, because the edges are dynamic. It's better to discuss the actual business need, but let's assume for now that instead of dynamic edges, we'll always just proceed into the next step (or steps) connected in the UI. It's simple to understand and subtracts features only from the most complex automations. Ok.

So the subsystem's UI now:
- assumes the user is logged in already and registered all the connections to social platforms via the main system:
  - entity User: id;
- shows the list of processes (with CRUD operations):
  - entity Process: id, userId;
  - entity Step: id, nextStepIds, processId, script (optional), type, userId (automatically set):
    - SendCommentStep: text, toPostId;
    - SendDmStep: text, toUserId;
    - StartStep;
    - StopStep;
    - WaitCommentStep: toPostId;
    - WaitDmStep;
- inside every process it shows a list of runs (with CRUD operations):
  - entity ProcessRun: id, lastStepRunId, processId, state;
  - entity StepRun: id, processRunId, status;
- a process is a drawing board to create nodes (Steps) and edges (step.nextStepIds);
- a step is a popup with script editor.

In the current version the might be multiple next steps (hence step.nextStepIds instead of step.nextStepId), but if we run them in parallel it becomes indeterministic who will change the processRun.state first and affects others. So we should either copy and paste processRun.state into every stepRun.state... actually it's the same as if we didn't create the processRun.state at all. Let's add stepRun.stateIn and stepRun.stateOut instead. Actually if we decided to allow a fanout operation, it's logical to add a fanin too (otherwise we're forced to have orphan states), and this means a stateRun might have multiple imcoming arrows, hence it should have an array of incoming states: stepRun.statesIn and single stepRun.stateOut. Also the next logical improvement is to add the EVERY and ANY gates as nodes. It's a regular node, which either waits for all incoming connections to succeed or just any of them. Ok, so instead of processRun.state (let's remove it but I'm not changing the text above for consistency) we now have stepRun.statesIn and stepRun.stateOut. Maybe the 'state' is not the best name now. Maybe it should be better called 'ins' and 'out', idk.

Well, it looks like we now have the UI, the ability to construct an automation process as a series of steps with an optional script inside any of them, which gets input objects as a parameter and returns an output object. These objects can contain arbitrary data (well, of course not that arbitrary, only the data we can actually serialize and store in the database, so no JavaScript functions and funny stuff). On receiving an event, which might be from the main system or from cron or whatever else we add later, or just from a timer, we select the processRuns in progress, the next stepRuns inside them, add these to a run queue and run inside a one-time and isolated Lambda, then gather the output and put it back into a database. Of course we create ProcessRuns from Processes and StepRuns from Steps somehow but it's an implementation detail. Looks like it should work. Of course we have to add some limits on objects size and on their serializability. Also we deliberately decided to not allow dynamic routing to next steps.

Maybe I missed something, but I'm writing this for almost 2 hours now, so let's call it a day from my side. Now please, review and tell me what you'd like to improve.

# Prompt #3

> Big issues

> 1. Scope: you’re building a workflow platform, not the feature.
> The prompt is “design the backend for comment → reply → DM → wait → DM.” A visual board, user scripts, Lambdas, fan-out/fan-in and ANY/EVERY gates are all extras. The grader wants to see judgment, and “I deliberately left these out and here’s the path to add them” scores higher than “I included everything.” My suggestion: declarative step types with config, and conditions on edges. Move scripts and gates to a “future extensions” paragraph. This also kills your whole sandboxing/limits/serialization discussion.

Sure, let's mention the basic version like you see it: "declarative step types with config, and conditions on edges". But it's too simplistic, I think the scripts are a very logical extension, it gives flexibility to be actually valueable automation platform, so let's not drop it. Basic version is basic, but the extension is my point.

> 2. Your no-dynamic-routing decision breaks the core use case.
> “Invalid email → re-ask” needs a branch or a loop. With only unconditional nextStepIds, you can’t express it. You introduced the if node and then retreated from it. Fix: edges carry an optional condition (contains, equals, matches email), evaluated by the engine. It’s deterministic and needs no scripts.

You're right, I haven't thought about the loops. Ok, let's allow the script to not jump steps arbitrarily, but an option to choose which outgoing connection to follow (like, the first if everything's fine, the second otherwise). Actually I think this functionality can be decoupled into an IF gate, which will be the only to have the "outcomes choosing" option, do you like it?

> 3. You’re missing the trigger and the correlation key.
> The example says “any post” and a keyword, so the trigger needs filters (keyword match, post scope), and it belongs on the Process, not in a WaitCommentStep(toPostId). More importantly, one process gets triggered by many commenters, so a run must be per (process, contact). The WaitDm step then correlates on (accountId, contactId), not just a user id. This is the heart of the “wait” mechanic, and the doc should say it explicitly.

Of course the trigger needs filters, that's why I mention that when we're waiting for a comment, we have to know the postId for it. It means our scheduler receives an event about a new comment for a post, then selects all stepRuns corresponding to WaitCommentSteps with the same toPostId and adds them all to the run queue. As a possible extension a user might want to follow all their posts at once, it would mean another type of Step, which waits by userId instead of postId. Ok?

> 4. You conflate two kinds of “user”.
> userId means the customer who owns the automation in some places and the commenter in others. Name them separately (Account/Tenant vs Contact) and put accountId on every row. That’s your real isolation boundary. Stripping toUserId from DMs is a weaker substitute.

Sure, naming might be better for sure. The idea was to not even show the user their id to not lure them into using it manually.

> 5. Reliability is missing entirely, and it’s what they’ll grade hardest.

> Webhooks are at-least-once, so you need dedupe on the platform event ID.

Sure, this might be the job of the main system, but we can add this into the subsystem too.

> A worker can crash between “sent DM” and “marked success,” and a retry would double-send. You need an explicit choice: record intent before sending, use idempotency keys where the platform allows, and decide at-least-once vs at-most-once for sends and justify it.

Well, yes, but I think you're overcomplicating already. It's a minor implementation detail.

> Claiming a waiting run must be atomic (UPDATE ... WHERE status='waiting' RETURNING, or FOR UPDATE SKIP LOCKED). Otherwise two DMs can advance one run twice.
> Run creation needs a unique key like (process, comment_id).

Fair point.

> Retries need backoff, a dead-letter path, and a split between permanent failures (blocked user, messaging window closed) and transient ones.

Sure, let's just mention it briefly.

> 6. Infinite loop trap: our own comment triggers us.
> Step 1 posts “Sent you a DM!” on the post, which generates a comment event. If that text matches a trigger, you’ve got a loop. Ignore events authored by the owning account. It’s a small point, but a sharp reviewer will notice.

Yep.

> Medium issues

> Waits need timeouts. Add an expiry deadline and an expired status, or you’ll accumulate orphan runs forever. Also: what if the user comments twice, or two automations are waiting on the same contact? Pick a rule, e.g. one active run per (process, contact), and say why.

Sure.

> Process versioning. What happens to in-flight runs when the customer edits the process? Runs should pin an immutable version (definition_version).

If we add a definition version, we should actually save all the definition versions. I think it's better to just create the ProcessRun from ProcessDef and all the StepRuns from StepDefs when the process kicks in. That way we can edit definitions in place.

> Naming. Process/ProcessRun and Step/StepRun are confusing. Definition vs Run reads better. Also add a waiting status. Your todo/running/failure/success can’t represent parked runs.

Ok, *Def and *Run, better? Please add the statuses you might see fit yourself.

> Queue or scan, pick one. Event-driven with a queue is the right call, so enqueue on event match. Use a next_run_at scan only for timers and expirations. “Constantly scans” will draw questions.

Event-based mode is the default of course. But I think we have to have a periodic scan for stale runs or to reconcile after a crash or something.

> Lambda for everything is overkill. Send steps are just calls to the existing send APIs, so plain workers do that. Lambda only makes sense if you keep scripts, another reason to cut them.

Ok.

> Limits should be concrete. Name per-account rate limits and quotas, max steps per run, and queue partitioning by account so one noisy tenant can’t starve others. “A hard limit” alone is too vague.

I don't think it's too vague, we're discussing the desing, not the implementation, right?

> Fan-in/ANY gates are a rabbit hole. ANY raises the question of what happens to the losing branches (cancel?), and parallel branches raise state-merge semantics. You already spent a paragraph on them. Cut.

Note this as needing more elaboration, but conceptually I think they are fine. By the way the concept is custom nodes with scripts, a few predefined gates (in-all, in-any, out-if/out-choose, start, stop) and explicitly custom-created edges without any logic. I don't like the idea of logic on edges (like "contains"), because it's too rigid.

> Smaller notes

> Cut the UI and CRUD section. The assignment says “backend system,” so the README should describe the API and engine, not the drawing board.

Ok, mention it briefly, no need to cut to zero.

> The Temporal/Step Functions paragraph is good, just shorter: “chose Postgres + queue to avoid the dependency; Temporal is the natural upgrade at scale.” That’s a justified decision in one line.

Fine.

> Worth verifying before you mention it: Instagram’s API restricts the first DM after a comment (a “private reply,” roughly one per comment within a limited window) and enforces a 24h messaging window afterward. The prompt says to assume sending works, so a brief note under considerations is plenty.

Don't bother, we're told to assume it works.
