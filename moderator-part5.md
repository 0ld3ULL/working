# Moderator Code - Part 5: Final Synthesis Prompts

## Final Synthesis for All Three Modes

```typescript
let synthesisPrompt;
const finalMode = session.moderatorMode || "collator";

// ==================== COLLATOR FINAL ====================
if (finalMode === "collator") {
  synthesisPrompt = `FINAL OUTPUT - COLLATOR MODE

PRIMARY DIRECTIVE: Deliver exactly what the user requested. Output ONLY the deliverable.

ORIGINAL USER REQUEST: ${session.question}

AI RESPONSES:
${allResponses}

Create the FINAL DELIVERABLE by combining the best outputs.
- User asked for ads? Output clean, ready-to-use ads
- User asked for writing? Output polished content
- User asked for a plan? Output actionable plan

NO meta-commentary. NO process notes. JUST the deliverable.`;

// ==================== ENQUIRER FINAL ====================
} else if (finalMode === "enquirer") {
  synthesisPrompt = `FINAL OUTPUT - ENQUIRER MODE

ORIGINAL USER REQUEST: ${session.question}

AI RESPONSES:
${allResponses}

YOUR TASK:
1. Brief "Assumptions/Notes" section (2-3 bullet points MAX)
2. Then the refined, polished final output

Keep notes very short - focus on the deliverable.`;

// ==================== DEVIL'S ADVOCATE FINAL ====================
} else {
  synthesisPrompt = `FINAL OUTPUT - DEVIL'S ADVOCATE MODE

ORIGINAL QUESTION: ${session.question}

AI RESPONSES + DELIBERATION:
${allResponses}
${deliberationSummary}

STRUCTURE:
## Answer
[Direct answer in 2-3 sentences]

## The Synthesis Journey
- Initial Positions: Key differences
- Deliberation Impact: How did discussion refine understanding?
- Points of Convergence: Where did AIs agree?

## Confidence & Consensus
- Overall Confidence: [High/Medium/Low]
- Degree of Consensus: [Strong/Partial/Contested]

## Key Insights
[Most valuable ideas from deliberation]

## Acknowledged Uncertainties
[What remains unclear]

## Actionable Recommendations
[Clear next steps]`;
}

const synthesizer = availableProviders[0];
const finalAnswer = await synthesizer.call(synthesisPrompt);## Mode Comparison Summary

| Aspect              | Collator                  | Enquirer                  | Devil's Advocate                    |
|---------------------|---------------------------|---------------------------|-------------------------------------|
| Goal                | Deliver user's request    | Light refinement          | Deep analysis                       |
| Moderator Style     | 1 brief question          | 1 clarifying question     | Structured analysis + tagged question |
| Debate Level        | None                      | Minimal                   | Full adversarial                    |
| Conclude Trigger    | "Ready to deliver"        | "Ready"                   | Confidence + remaining uncertainty  |
| Respondent Focus    | Improve deliverable       | Answer + update           | Position + confidence + engagement  |
| Final Synthesis     | Just the deliverable      | Brief notes + deliverable | Full structured analysis            |
| Best For            | Creative tasks, ads, copy | Unclear requirements      | Research, strategy, decisions       |
