# Moderator Full Code Reference

Complete code for all three Moderator modes (Collator, Enquirer, Devil's Advocate) including selection, prompts, respondent handling, and final synthesis.

---

## Part 1: Think Tank Endpoint & Moderator Selection

```typescript
// Think Tank deliberation - multi-round AI dialogue - protected
// Runs in BACKGROUND so user can navigate away without losing progress
app.post("/api/brainstorm/thinktank/:sessionId", isAuthenticated, async (req, res) => {
  const sessionId = parseInt(req.params.sessionId);
  const userId = (req.session as any).userId;
  const requestedModerator = req.body?.moderator; // Optional: user can override the stored moderator
  const session = await storage.getBrainstormSession(sessionId, userId);

  if (!session || !session.round1Responses) {
    return res.status(404).json({ error: "Session not found or Round 1 not completed" });
  }

  // Check if Think Tank is already in progress or completed
  if (session.thinkTank && session.thinkTank.length > 0) {
    return res.json({ 
      sessionId, 
      status: "already_completed",
      deliberationRounds: session.thinkTank,
      message: "Think Tank already completed for this session"
    });
  }

  // Get all providers at thinktank tier (highest capability) - user-scoped
  const providers = await getActiveProviders(userId, "thinktank");
  
  // Choose moderator AI - priority: request body > session stored preference > default order
  let moderator;
  const moderatorToUse = requestedModerator || session.selectedModerator;
  
  if (moderatorToUse) {
    // User specified a moderator (either in request or stored in session)
    moderator = providers.find(p => p.name === moderatorToUse);
    if (!moderator) {
      return res.status(400).json({ error: `Requested moderator "${moderatorToUse}" is not available` });
    }
    console.log(`Using moderator: ${moderatorToUse} (${requestedModerator ? 'from request' : 'from session'})`);
  } else {
    // Default priority order (prefer Claude, then GPT, then others)
    const moderatorOrder = ["Claude", "ChatGPT", "Gemini", "DeepSeek", "Grok"];
    moderator = moderatorOrder
      .map(name => providers.find(p => p.name === name))
      .filter(Boolean)[0];
    console.log(`Using default moderator: ${moderator?.name}`);
  }

  if (!moderator) {
    return res.status(500).json({ error: "No AI provider available for moderating" });
  }

  // Mark Think Tank as running in the database (persists across refresh)
  await storage.updateBrainstormSession(sessionId, userId, {
    thinkTankStatus: "running",
  });

  // Immediately return response - process runs in background
  res.json({ 
    sessionId, 
    status: "started",
    moderator: moderator.name,
    message: "Think Tank started. You can safely navigate away - results will be saved automatically."
  });

  // Run the Think Tank process in background (not awaited)
  runThinkTankInBackground(sessionId, session, providers, moderator, userId).catch(async (err) => {
    console.error(`Background Think Tank failed for session ${sessionId}:`, err);
    emitThinkTankProgress({ sessionId, step: "error", status: "error", error: err.message });
    // Reset status on error so user can retry
    await storage.updateBrainstormSession(sessionId, userId, {
      thinkTankStatus: "idle",
    });
  });
});// Background Think Tank processing function
async function runThinkTankInBackground(
  sessionId: number, 
  session: any, 
  providers: any[], 
  moderator: any,
  userId: string
) {
  // Emit initial "sent" progress
  emitThinkTankProgress({ sessionId, step: "sent", status: "active" });

  const MAX_ROUNDS = 3;
  const deliberationRounds: Array<{
    round: number;
    moderatorQuestion: string;
    responses: Record<string, string>;
  }> = [];

  // Mark "sent" complete
  emitThinkTankProgress({ sessionId, step: "sent", status: "complete" });

  // Build context from Round 1 responses
  let conversationContext = `ORIGINAL QUESTION: ${session.question}\n\nINITIAL RESPONSES:\n${Object.entries(session.round1Responses)
    .map(([name, response]) => `=== ${name} ===\n${response}`)
    .join("\n\n")}`;

  // Run deliberation rounds
  for (let roundNum = 1; roundNum <= MAX_ROUNDS; roundNum++) {
    // Emit progress for current round
    const stepMap: Record<number, "q1" | "q2" | "q3"> = { 1: "q1", 2: "q2", 3: "q3" };
    emitThinkTankProgress({ sessionId, step: stepMap[roundNum], status: "active" });
    
    // Step 1: Moderator generates a diagnostic analysis and follow-up question
    const moderatorPrompt = roundNum === 1
      ? `You are the Moderator AI in a Think Tank session. Your goal is to deepen collective understanding by forcing scrutiny of critical junctures.

${conversationContext}

PERFORM THIS ANALYSIS:

1. **DISAGREEMENT MATRIX**: List specific claims where responses contradict (distinguish: factual disputes vs. value differences vs. definitional disagreements)

2. **UNEXAMINED PILLARS**: Identify 1-2 assumptions that multiple responses rely on but none have questioned

3. **STRATEGIC GAP**: Find important perspectives, consequences, or edge cases missing entirely from all responses

4. **CONFIDENCE SPREAD**: Where do the AIs seem most certain? Least certain? Any overconfidence on weak foundations?

QUESTION SELECTION (choose ONE from this priority order):
- [STRESS TEST]: Challenge the strongest-held shared assumption - push where consensus might be wrong
- [CLARIFY CONFLICT]: Force direct address of a specific factual or interpretive disagreement
- [FILL GAP]: Probe an important unexplored dimension or consequence
- [PUSH UNCERTAINTY]: Explore scenarios where the consensus view fails or breaks down

FORMAT YOUR RESPONSE:
First, provide your brief diagnostic analysis (2-3 sentences covering the most important finding).
Then tag and ask your question: "[TAG]: Your question here"

If genuine consensus exists with no significant gaps worth exploring, respond with:
"CONCLUDE: [reason] | CONFIDENCE: [H/M/L] | REMAINING UNCERTAINTY: [what questions remain unanswered]"

Remember: Your goal is not just to extend discussion, but to expose weaknesses, deepen understanding, and push toward truth.`
      : `You are the Moderator AI continuing a Think Tank deliberation. Your goal is to deepen collective understanding by forcing scrutiny of critical junctures.

${conversationContext}

Previous deliberation rounds:
${deliberationRounds.map(r => `Round ${r.round}:\nModerator asked: ${r.moderatorQuestion}\nResponses:\n${Object.entries(r.responses).map(([name, resp]) => `${name}: ${resp}`).join("\n")}`).join("\n\n")}

EVALUATE THIS ROUND:

1. **POSITION EVOLUTION**: Which AIs shifted positions? Which held firm? Were the shifts substantive or cosmetic?

2. **RESOLUTION STATUS**: Which disagreements were resolved? Which remain? Any new disagreements that emerged?

3. **DEPTH CHECK**: Has the discussion genuinely deepened, or are AIs repeating themselves with different words?

4. **CONVERGENCE QUALITY**: Is consensus emerging from genuine persuasion or from fatigue/social pressure?

ESCALATION PRINCIPLE: If Round 1 was exploratory, Round 2+ should be MORE adversarial. Ask the question most likely to BREAK emerging consensus IF it's wrong.

QUESTION SELECTION (choose ONE):
- [STRESS TEST]: Target the weakest point of the emerging consensus
- [CLARIFY CONFLICT]: Resolve a specific remaining disagreement  
- [FILL GAP]: Address a critical blind spot
- [PUSH UNCERTAINTY]: "Under what conditions would the consensus position fail?"

FORMAT YOUR RESPONSE:
Brief analysis of what changed this round (1-2 sentences).
Then tag and ask: "[TAG]: Your question here"

If discussion has reached genuine depth with substantive consensus, respond with:
"CONCLUDE: [reason] | CONFIDENCE: [H/M/L] | REMAINING UNCERTAINTY: [what's still unknown]"

Important: This is round ${roundNum} of ${MAX_ROUNDS}. Conclude if consensus is genuine, but don't rush - depth matters more than speed.`;    try {
      const moderatorResponse = await moderator.call(moderatorPrompt);

      // Check if moderator wants to conclude
      if (moderatorResponse.trim().startsWith("CONCLUDE:")) {
        console.log(`Think Tank concluding after round ${roundNum - 1}: ${moderatorResponse}`);
        break;
      }

      // Step 2: All AIs respond to the moderator's question
      const followUpResponses: Record<string, string> = {};
      
      const respondingProviders = providers.filter(p => p.name !== moderator.name);
      
      const responsePromises = respondingProviders.map(async (provider) => {
        // Rotating cognitive mandates based on round number
        const cognitiveMandates = [
          "Steel-man the opposing view: Before critiquing any AI's position, first present their strongest possible version of their argument.",
          "Identify weakness: What is the single weakest point in your own current position? Be genuinely self-critical.",
          "Falsifiability: What specific evidence or scenario would convincingly change your mind on this question?"
        ];
        const mandate = cognitiveMandates[(roundNum - 1) % cognitiveMandates.length];

        const respondentPrompt = `You are ${provider.name} in Round ${roundNum} of a Think Tank deliberation.

MODERATOR'S QUESTION: "${moderatorResponse}"

YOUR CONTEXT:
Original question: ${session.question}

Your initial position: ${session.round1Responses![provider.name] || "You did not provide an initial response."}

${roundNum > 1 ? `Previous rounds:\n${deliberationRounds.map(r => `Round ${r.round}:\n- Moderator asked: ${r.moderatorQuestion}\n- Your response: ${r.responses[provider.name] || "N/A"}`).join("\n\n")}` : ""}

Other AIs' current positions (for reference):
${Object.entries(session.round1Responses || {}).filter(([name]) => name !== provider.name).map(([name, response]) => `${name}: ${String(response).slice(0, 300)}...`).join("\n")}

RESPONSE REQUIREMENTS:

1. **DIRECT ENGAGEMENT** (required):
   Identify the single strongest point from another AI that challenges your position.
   Format: "[AI Name]'s claim that [specific claim] challenges my view because..."
   Then explain HOW it exposes a potential weakness in your argument.

2. **POSITION STATEMENT** (required):
   If your position changed: "POSITION SHIFT: Previously [X], now [Y] because [specific argument that convinced you]"
   If maintained: "POSITION HELD: I maintain [X] despite [challenge] because [your reasoning]"

3. **CONFIDENCE & UNCERTAINTY** (required):
   Format: "CONFIDENCE: [High/Medium/Low] because [brief justification]"
   Then: "KEY UNCERTAINTY: [What would change your view]"

4. **ASSUMPTION DISCLOSURE** (required):
   Format: "MY ANSWER ASSUMES: [critical assumption]. If this is wrong, my conclusion would change to [X]"

5. **COGNITIVE MANDATE FOR THIS ROUND**:
   ${mandate}

CONSTRAINTS:
- Take substantive positions - don't hedge everything
- Steelman opposing views before critiquing them
- Be genuinely open to changing your mind if persuaded
- Credit good arguments from others explicitly
- Provide thorough, genius-level analysis - use the space you need`;

        try {
          const response = await provider.call(respondentPrompt);
          return { name: provider.name, response };
        } catch (error: any) {
          return { name: provider.name, response: `Error: ${error.message}` };
        }
      });

      const responses = await Promise.all(responsePromises);
      responses.forEach(({ name, response }) => {
        followUpResponses[name] = response;
      });

      deliberationRounds.push({
        round: roundNum,
        moderatorQuestion: moderatorResponse,
        responses: followUpResponses,
      });

      // Mark this round complete
      emitThinkTankProgress({ sessionId, step: stepMap[roundNum], status: "complete" });

    } catch (error: any) {
      console.error(`Think Tank round ${roundNum} failed:`, error.message);
      break;
    }
  }

  // Emit summarising progress
  emitThinkTankProgress({ sessionId, step: "summarising", status: "active" });

  // Save deliberation rounds to session and mark as completed
  await storage.updateBrainstormSession(sessionId, userId, {
    thinkTank: deliberationRounds,
    thinkTankStatus: "completed",
  });

  // Mark summarising and overall complete
  emitThinkTankProgress({ sessionId, step: "summarising", status: "complete" });
  emitThinkTankProgress({ sessionId, step: "complete", status: "complete" });

  // Record Think Tank usage for billing ($1 premium per session)
  if (userId) {
    try {
      const user = await storage.getUser(userId);
      if (user?.stripeCustomerId) {
        const usageResult = await stripeService.recordThinkTankUsage(user.stripeCustomerId, sessionId);
        if (!usageResult.success) {
          console.log(`Think Tank usage recording skipped: ${usageResult.error}`);
        }
      }
    } catch (error: any) {
      console.error('Failed to record Think Tank usage:', error.message);
    }
  }

  console.log(`Think Tank background process completed for session ${sessionId}`);
}// ========== STEP 2: Think Tank (if thinktank tier and not already done) ==========
if (isThinkTank && (!session.thinkTank || session.thinkTank.length === 0)) {
  emitAutoRunProgress({ sessionId, stage: "thinktank", status: "active" });
  
  const providers = await getActiveProviders(userId, "thinktank");
  
  // Find moderator
  let moderator;
  if (moderatorName) {
    moderator = providers.find((p: any) => p.name === moderatorName);
  }
  if (!moderator) {
    const moderatorOrder = ["Claude", "ChatGPT", "Gemini", "DeepSeek", "Grok"];
    moderator = moderatorOrder
      .map(name => providers.find((p: any) => p.name === name))
      .filter(Boolean)[0];
  }
  
  if (!moderator) {
    throw new Error("No AI provider available for moderating");
  }
  
  // Mark Think Tank as running
  await storage.updateBrainstormSession(sessionId, userId, {
    thinkTankStatus: "running",
  });
  
  // Run Think Tank (same logic as the standalone endpoint)
  const MAX_ROUNDS = 3;
  const deliberationRounds: Array<{
    round: number;
    moderatorQuestion: string;
    responses: Record<string, string>;
  }> = [];

  let conversationContext = `ORIGINAL QUESTION: ${session.question}\n\nINITIAL RESPONSES:\n${Object.entries(session.round1Responses)
    .map(([name, response]) => `=== ${name} ===\n${response}`)
    .join("\n\n")}`;

  const sessionModeratorMode = (session.moderatorMode as any) || "collator";
  
  for (let roundNum = 1; roundNum <= MAX_ROUNDS; roundNum++) {
    emitAutoRunProgress({ sessionId, stage: "thinktank", step: `q${roundNum}`, status: "active" });
    
    let moderatorPrompt: string;
    
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

Focus on DELIVERING what the user wants. Keep to 1 sentence. No debate or critique needed.
If the deliverable is ready, respond with: "CONCLUDE: Ready to deliver"`
        : `You are the Moderator helping refine the output. Round ${roundNum}.

Previous refinements:
${deliberationRounds.map(r => `Q: ${r.moderatorQuestion}\nResponses: ${Object.entries(r.responses).map(([n, resp]) => `${n}: ${resp}`).join(", ")}`).join("\n")}

Ask ONE question to further improve the output, or "CONCLUDE: Ready to deliver" if complete.`;
    
    // ==================== ENQUIRER MODE ====================
    } else if (sessionModeratorMode === "enquirer") {
      moderatorPrompt = roundNum === 1
        ? `You are the Moderator guiding a light refinement. Round ${roundNum}.

ORIGINAL USER REQUEST: ${session.question}

CURRENT RESPONSES:
${Object.entries(session.round2Responses || session.round1Responses).map(([name, resp]) => `${name}:\n${resp}`).join("\n\n---\n\n")}

Ask ONE clarifying question that helps:
- Tighten requirements or assumptions
- Pick between close alternatives
- Refine the output

Keep it friendly and constructive (1 sentence). Focus on improvement, not debate.
If done, respond with: "CONCLUDE: Ready"`
        : `You are the Moderator continuing refinement. Round ${roundNum}.

Previous clarifications:
${deliberationRounds.map(r => `Q: ${r.moderatorQuestion}`).join("\n")}

Ask ONE more clarifying question, or "CONCLUDE: Ready" if the output is polished.`;
    
    // ==================== DEVIL'S ADVOCATE MODE ====================
    } else {
      moderatorPrompt = roundNum === 1
        ? `You are the Moderator AI in a Think Tank session. Your goal is to deepen collective understanding by forcing scrutiny of critical junctures.

${conversationContext}

PERFORM THIS ANALYSIS:

1. **DISAGREEMENT MATRIX**: List specific claims where responses contradict
2. **UNEXAMINED PILLARS**: Identify 1-2 assumptions that multiple responses rely on but none have questioned
3. **STRATEGIC GAP**: Find important perspectives missing entirely from all responses
4. **CONFIDENCE SPREAD**: Where do the AIs seem most certain? Least certain?

QUESTION SELECTION (choose ONE):
- [STRESS TEST]: Challenge the strongest-held shared assumption
- [CLARIFY CONFLICT]: Force direct address of a specific disagreement
- [FILL GAP]: Probe an important unexplored dimension
- [PUSH UNCERTAINTY]: Explore scenarios where the consensus fails

FORMAT YOUR RESPONSE:
Brief diagnostic analysis (2-3 sentences).
Then tag and ask your question: "[TAG]: Your question here"

If genuine consensus exists, respond with:
"CONCLUDE: [reason] | CONFIDENCE: [H/M/L] | REMAINING UNCERTAINTY: [what remains]"`
        : `You are the Moderator AI continuing Think Tank deliberation.

${conversationContext}

Previous rounds:
${deliberationRounds.map(r => `Round ${r.round}:\nModerator asked: ${r.moderatorQuestion}\nResponses:\n${Object.entries(r.responses).map(([name, resp]) => `${name}: ${resp}`).join("\n")}`).join("\n\n")}

ESCALATION: Round 2+ should be MORE adversarial. Ask the question most likely to BREAK emerging consensus IF it's wrong.

Format: Brief analysis, then "[TAG]: Your question"
Or if done: "CONCLUDE: [reason] | CONFIDENCE: [H/M/L] | REMAINING UNCERTAINTY: [what's still unknown]"`;
    }    try {
      const moderatorResponse = await moderator.call(moderatorPrompt);

      if (moderatorResponse.trim().startsWith("CONCLUDE:")) {
        break;
      }

      const respondingProviders = providers.filter((p: any) => p.name !== moderator.name);
      const roundResponses: Record<string, string> = {};
      
      const responsePromises = respondingProviders.map(async (provider: any) => {
        let respondentPrompt: string;
        
        // ==================== COLLATOR RESPONDENT ====================
        if (sessionModeratorMode === "collator") {
          respondentPrompt = `SYNTHESIS REFINEMENT - Round ${roundNum}

The moderator asked: ${moderatorResponse}

ORIGINAL USER REQUEST: ${session.question}

YOUR PREVIOUS RESPONSE: ${session.round2Responses?.[provider.name] || session.round1Responses![provider.name] || "N/A"}

Respond directly with concrete improvements to the deliverable. Be concise and practical. No debate or analysis - just improve the output.`;
        
        // ==================== ENQUIRER RESPONDENT ====================
        } else if (sessionModeratorMode === "enquirer") {
          respondentPrompt = `CLARIFICATION RESPONSE - Round ${roundNum}

The moderator asked: ${moderatorResponse}

ORIGINAL USER REQUEST: ${session.question}

YOUR PREVIOUS RESPONSE: ${session.round2Responses?.[provider.name] || session.round1Responses![provider.name] || "N/A"}

Briefly answer the clarification and update your deliverable. Keep it concise and user-focused.`;
        
        // ==================== DEVIL'S ADVOCATE RESPONDENT ====================
        } else {
          respondentPrompt = `You are ${provider.name} in Round ${roundNum} of Think Tank deliberation.

MODERATOR'S QUESTION: "${moderatorResponse}"

Original question: ${session.question}
Your initial position: ${session.round1Responses![provider.name] || "N/A"}

RESPONSE REQUIREMENTS:
1. **DIRECT ENGAGEMENT**: Identify the strongest point from another AI that challenges your position
2. **POSITION STATEMENT**: State if position changed or held, with reasoning
3. **CONFIDENCE**: [High/Medium/Low] with justification
4. **KEY UNCERTAINTY**: What would change your view

Be substantive but focused (150-250 words).`;
        }

        try {
          const response = await provider.call(respondentPrompt);
          return { name: provider.name, response };
        } catch (error: any) {
          return { name: provider.name, response: `Error: ${error.message}` };
        }
      });

      const responses = await Promise.all(responsePromises);
      responses.forEach(({ name, response }: any) => {
        roundResponses[name] = response;
      });

      deliberationRounds.push({
        round: roundNum,
        moderatorQuestion: moderatorResponse,
        responses: roundResponses,
      });

      conversationContext += `\n\n=== DELIBERATION ROUND ${roundNum} ===\nModerator Question: ${moderatorResponse}\nResponses:\n${Object.entries(roundResponses).map(([name, resp]) => `${name}: ${resp}`).join("\n\n")}`;

    } catch (error: any) {
      console.error(`Auto-run Think Tank round ${roundNum} failed:`, error.message);
      break;
    }
  }

  // Save Think Tank results
  await storage.updateBrainstormSession(sessionId, userId, {
    thinkTank: deliberationRounds,
    thinkTankStatus: "completed",
  });
  
  session.thinkTank = deliberationRounds;
  
  // Record Think Tank usage for billing
  try {
    const user = await storage.getUser(userId);
    if (user?.stripeCustomerId) {
      await stripeService.recordThinkTankUsage(user.stripeCustomerId, sessionId);
    }
  } catch (error: any) {
    console.error('Failed to record Think Tank usage:', error.message);
  }
  
  emitAutoRunProgress({ sessionId, stage: "thinktank", status: "complete" });
} else if (isThinkTank) {
  emitAutoRunProgress({ sessionId, stage: "thinktank", status: "complete" });
}// ========== STEP 3: Generate Final Answer with mode-specific prompts ==========
emitAutoRunProgress({ sessionId, stage: "final", status: "active" });

let synthesisPrompt: string;
const finalMode = (session.moderatorMode as any) || "collator";

// ==================== COLLATOR FINAL SYNTHESIS ====================
if (finalMode === "collator") {
  // Collator mode - just deliver the final output, no meta-commentary
  const allResponses = Object.entries(session.round2Responses || session.round1Responses || {})
    .map(([name, response]) => `${name}:\n${response}`)
    .join("\n\n---\n\n");
    
  synthesisPrompt = `FINAL OUTPUT - COLLATOR MODE

PRIMARY DIRECTIVE: Deliver exactly what the user requested. Output the final deliverable ONLY.

ORIGINAL USER REQUEST: ${session.question}

AI RESPONSES:
${allResponses}
${session.thinkTank && session.thinkTank.length > 0 ? `\n\nREFINEMENTS:\n${session.thinkTank.map((r: any) => `Q: ${r.moderatorQuestion}\nResponses: ${Object.entries(r.responses).map(([n, resp]) => `${n}: ${resp}`).join("\n")}`).join("\n\n")}` : ""}

YOUR TASK:
Create the FINAL DELIVERABLE by combining the best outputs from all AIs.
- If the user asked for ads: output a clean, ready-to-use list of ads
- If the user asked for writing: output the polished content
- If the user asked for a plan: output the actionable plan

FORMAT RULES:
- NO meta-commentary or process notes
- NO "here's what I learned from other AIs"
- NO consensus/disagreement analysis
- JUST the final, polished deliverable the user can use immediately`;

// ==================== ENQUIRER FINAL SYNTHESIS ====================
} else if (finalMode === "enquirer") {
  // Enquirer mode - brief notes then deliverable
  const allResponses = Object.entries(session.round2Responses || session.round1Responses || {})
    .map(([name, response]) => `${name}:\n${response}`)
    .join("\n\n---\n\n");
    
  synthesisPrompt = `FINAL OUTPUT - ENQUIRER MODE

ORIGINAL USER REQUEST: ${session.question}

AI RESPONSES:
${allResponses}
${session.thinkTank && session.thinkTank.length > 0 ? `\n\nCLARIFICATIONS:\n${session.thinkTank.map((r: any) => `Q: ${r.moderatorQuestion}\nResponses: ${Object.entries(r.responses).map(([n, resp]) => `${n}: ${resp}`).join("\n")}`).join("\n\n")}` : ""}

YOUR TASK:
1. Start with a brief "Assumptions/Notes" section (2-3 bullet points MAX)
2. Then deliver the refined, polished final output

Keep the notes section very short - the focus is on the deliverable.`;// ==================== DEVIL'S ADVOCATE FINAL SYNTHESIS (WITH THINK TANK) ====================
} else if (isThinkTank && session.thinkTank && session.thinkTank.length > 0) {
  // Advocate mode with Think Tank
  const deliberationSummary = session.thinkTank.map((round: any) => 
    `=== Deliberation Round ${round.round} ===\nModerator Question: "${round.moderatorQuestion}"\n\nResponses:\n${Object.entries(round.responses).map(([name, resp]) => `${name}: ${resp}`).join("\n\n")}`
  ).join("\n\n---\n\n");

  synthesisPrompt = `You are the High Level Synthesis AI. Your task is to analyze the entire Think Tank deliberation and create a definitive, comprehensive final answer.

ORIGINAL QUESTION:
${session.question}

INITIAL RESPONSES (Round 1):
${Object.entries(session.round1Responses || {})
  .map(([name, response]) => `=== ${name} ===\n${response}`)
  .join("\n\n")}

THINK TANK DELIBERATION:
${deliberationSummary}

YOUR SYNTHESIS MUST FOLLOW THIS STRUCTURE:

## Answer
[Direct, actionable answer to the question in 2-3 sentences - the core conclusion]

## The Synthesis Journey
- **Initial Positions**: Key differences in initial approaches
- **Deliberation Impact**: How did the Think Tank process refine understanding?
- **Points of Convergence**: Where did the AIs reach agreement?

## Confidence & Consensus
- **Overall Confidence**: [High/Medium/Low] - [justification]
- **Degree of Consensus**: [Strong/Partial/Contested]

## Key Insights
[The most valuable ideas that emerged from deliberation]

## Acknowledged Uncertainties
- What remains unclear or debatable
- Conditions where this answer might not apply

## Actionable Recommendations
[Clear, practical next steps based on the synthesis]`;

// ==================== DEVIL'S ADVOCATE FINAL SYNTHESIS (STANDARD ROUND 2) ====================
} else {
  // Advocate mode standard Round 2 synthesis
  synthesisPrompt = `You are the High Level Synthesis AI. Your task is to analyze all Round 2 responses and create a definitive, comprehensive final answer.

ORIGINAL QUESTION:
${session.question}

ROUND 1 RESPONSES:
${Object.entries(session.round1Responses || {})
  .map(([name, response]) => `=== ${name} ===\n${response}`)
  .join("\n\n")}

ROUND 2 RESPONSES:
${Object.entries(session.round2Responses!)
  .map(([name, response]) => `=== ${name} ===\n${response}`)
  .join("\n\n")}

YOUR SYNTHESIS MUST FOLLOW THIS STRUCTURE:

## Answer
[Direct, actionable answer in 2-3 sentences]

## The Synthesis Journey
- **Initial Diversity**: Key differences in Round 1 approaches
- **Cross-Pollination Impact**: How did AIs modify their views?
- **Points of Convergence**: Where did multiple AIs arrive at similar conclusions?

## Confidence & Consensus
- **Overall Confidence**: [High/Medium/Low] - [justification]
- **Degree of Consensus**: [Strong/Partial/Contested]

## Key Insights
[Most valuable ideas from collective analysis]

## Acknowledged Uncertainties
- What remains unclear
- What additional information would help

## Actionable Recommendations
[Clear, practical next steps]`;
}

// Use HIGH tier for final synthesis
const currentProviders = await getActiveProviders(userId, "high");
const preferredOrder = ["Claude", "ChatGPT", "Gemini", "DeepSeek", "Grok"];
const availableProviders = preferredOrder
  .map(name => currentProviders.find((p: any) => p.name === name))
  .filter(Boolean);

if (availableProviders.length === 0) {
  throw new Error("No AI provider available for synthesis");
}

const synthesizer = availableProviders[0];
const finalAnswer = await synthesizer.call(synthesisPrompt);## Mode Comparison Summary
| Aspect | Collator | Enquirer | Devil's Advocate |
|--------|----------|----------|------------------|
| **Goal** | Deliver user's request | Light refinement | Deep analysis |
| **Moderator Style** | 1 brief question | 1 clarifying question | Structured analysis + tagged question |
| **Debate Level** | None | Minimal | Full adversarial |
| **Conclude Trigger** | "Ready to deliver" | "Ready" | Confidence + remaining uncertainty |
| **Respondent Focus** | Improve deliverable | Answer + update | Position + confidence + engagement |
| **Final Synthesis** | Just the deliverable | Brief notes + deliverable | Full structured analysis |
| **Best For** | Creative tasks, ads, copy | Unclear requirements | Research, strategy, decisions |
