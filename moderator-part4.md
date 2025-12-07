# Moderator Code - Part 4: Respondent Prompts & Final Synthesis

## Respondent Prompts for All Three Modes

```typescript
// Inside the round loop, after getting moderator response
const respondingProviders = providers.filter(p => p.name !== moderator.name);

const responsePromises = respondingProviders.map(async (provider) => {
  let respondentPrompt;
  
  // ==================== COLLATOR RESPONDENT ====================
  if (sessionModeratorMode === "collator") {
    respondentPrompt = `SYNTHESIS REFINEMENT - Round ${roundNum}

Moderator asked: ${moderatorResponse}

ORIGINAL USER REQUEST: ${session.question}
YOUR PREVIOUS RESPONSE: ${session.round1Responses[provider.name] || "N/A"}

Respond with concrete improvements to the deliverable. Be concise. No debate - just improve.`;
  
  // ==================== ENQUIRER RESPONDENT ====================
  } else if (sessionModeratorMode === "enquirer") {
    respondentPrompt = `CLARIFICATION RESPONSE - Round ${roundNum}

Moderator asked: ${moderatorResponse}

ORIGINAL USER REQUEST: ${session.question}
YOUR PREVIOUS RESPONSE: ${session.round1Responses[provider.name] || "N/A"}

Briefly answer the clarification and update your deliverable. Keep it user-focused.`;
  
  // ==================== DEVIL'S ADVOCATE RESPONDENT ====================
  } else {
    respondentPrompt = `You are ${provider.name} in Round ${roundNum} of Think Tank.

MODERATOR'S QUESTION: "${moderatorResponse}"

Original question: ${session.question}
Your initial position: ${session.round1Responses[provider.name] || "N/A"}

REQUIREMENTS:
1. DIRECT ENGAGEMENT: Strongest challenging point from another AI
2. POSITION STATEMENT: Changed or held, with reasoning
3. CONFIDENCE: [High/Medium/Low] with justification
4. KEY UNCERTAINTY: What would change your view

Be substantive (150-250 words).`;
  }

  const response = await provider.call(respondentPrompt);
  return { name: provider.name, response };
});
