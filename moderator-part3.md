# Moderator Code - Part 3: All Three Moderator Mode Prompts

## Collator, Enquirer, and Devil's Advocate Mode Prompts

```typescript
// Inside Auto-Run Think Tank loop
const sessionModeratorMode = session.moderatorMode || "collator";

for (let roundNum = 1; roundNum <= MAX_ROUNDS; roundNum++) {
  let moderatorPrompt;
  
  // ==================== COLLATOR MODE ====================
  if (sessionModeratorMode === "collator") {
    moderatorPrompt = roundNum === 1
      ? `You are the Moderator helping synthesize responses. Round ${roundNum}.

ORIGINAL USER REQUEST: ${session.question}

CURRENT RESPONSES:
${Object.entries(session.round2Responses || session.round1Responses).map(([name, resp]) => `${name}:\n${resp}`).join("\n\n---\n\n")}

Ask ONE brief question that helps:
- Fill any gaps in the deliverable
- Choose between similar good options
- Improve the final output quality

Focus on DELIVERING what the user wants. Keep to 1 sentence. No debate needed.
If ready, respond: "CONCLUDE: Ready to deliver"`
      : `Moderator continuing refinement. Round ${roundNum}.

Previous refinements:
${deliberationRounds.map(r => `Q: ${r.moderatorQuestion}`).join("\n")}

Ask ONE question to improve output, or "CONCLUDE: Ready to deliver" if complete.`;
  
  // ==================== ENQUIRER MODE ====================
  } else if (sessionModeratorMode === "enquirer") {
    moderatorPrompt = roundNum === 1
      ? `You are the Moderator guiding light refinement. Round ${roundNum}.

ORIGINAL USER REQUEST: ${session.question}

CURRENT RESPONSES:
${Object.entries(session.round2Responses || session.round1Responses).map(([name, resp]) => `${name}:\n${resp}`).join("\n\n---\n\n")}

Ask ONE clarifying question that helps:
- Tighten requirements or assumptions
- Pick between close alternatives
- Refine the output

Friendly and constructive (1 sentence). Focus on improvement.
If done: "CONCLUDE: Ready"`
      : `Moderator continuing. Round ${roundNum}.

Previous clarifications:
${deliberationRounds.map(r => `Q: ${r.moderatorQuestion}`).join("\n")}

Ask ONE clarifying question, or "CONCLUDE: Ready" if polished.`;
  
  // ==================== DEVIL'S ADVOCATE MODE ====================
  } else {
    moderatorPrompt = roundNum === 1
      ? `You are the Moderator AI in Think Tank. Deepen understanding by forcing scrutiny.

${conversationContext}

ANALYSIS:
1. DISAGREEMENT MATRIX: Where do responses contradict?
2. UNEXAMINED PILLARS: Assumptions none questioned?
3. STRATEGIC GAP: Missing perspectives?
4. CONFIDENCE SPREAD: Overconfidence anywhere?

QUESTION (choose ONE):
- [STRESS TEST]: Challenge shared assumption
- [CLARIFY CONFLICT]: Force address of disagreement
- [FILL GAP]: Probe unexplored dimension
- [PUSH UNCERTAINTY]: Where does consensus fail?

Format: Brief analysis (2-3 sentences), then "[TAG]: Question"
Or: "CONCLUDE: [reason] | CONFIDENCE: [H/M/L] | REMAINING UNCERTAINTY: [what remains]"`
      : `Moderator continuing Think Tank. Round ${roundNum}.

Previous rounds:
${deliberationRounds.map(r => `Round ${r.round}: ${r.moderatorQuestion}`).join("\n")}

ESCALATION: Be MORE adversarial. Break consensus if it's wrong.
Format: Brief analysis, then "[TAG]: Question"
Or: "CONCLUDE: [reason] | CONFIDENCE: [H/M/L] | REMAINING UNCERTAINTY: [unknown]"`;
  }
}
