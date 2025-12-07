# Moderator Code - Part 2: Respondent Prompts & Round Completion

## Respondent Prompts (Devil's Advocate Mode)

```typescript
async function runThinkTankInBackground(sessionId, session, providers, moderator, userId) {
  emitThinkTankProgress({ sessionId, step: "sent", status: "active" });

  const MAX_ROUNDS = 3;
  const deliberationRounds = [];

  emitThinkTankProgress({ sessionId, step: "sent", status: "complete" });

  let conversationContext = `ORIGINAL QUESTION: ${session.question}\n\nINITIAL RESPONSES:\n${Object.entries(session.round1Responses)
    .map(([name, response]) => `=== ${name} ===\n${response}`)
    .join("\n\n")}`;

  for (let roundNum = 1; roundNum <= MAX_ROUNDS; roundNum++) {
    const stepMap = { 1: "q1", 2: "q2", 3: "q3" };
    emitThinkTankProgress({ sessionId, step: stepMap[roundNum], status: "active" });
    
    // Moderator prompt logic shown in Part 1...
    
    try {
      const moderatorResponse = await moderator.call(moderatorPrompt);

      if (moderatorResponse.trim().startsWith("CONCLUDE:")) {
        break;
      }

      const followUpResponses = {};
      const respondingProviders = providers.filter(p => p.name !== moderator.name);
      
      const responsePromises = respondingProviders.map(async (provider) => {
        const cognitiveMandates = [
          "Steel-man the opposing view before critiquing.",
          "Identify the weakest point in your own position.",
          "What evidence would change your mind?"
        ];
        const mandate = cognitiveMandates[(roundNum - 1) % 3];

        const respondentPrompt = `You are ${provider.name} in Round ${roundNum} of Think Tank.

MODERATOR'S QUESTION: "${moderatorResponse}"

Original question: ${session.question}
Your initial position: ${session.round1Responses[provider.name] || "N/A"}

RESPONSE REQUIREMENTS:
1. DIRECT ENGAGEMENT: Identify strongest challenging point from another AI
2. POSITION STATEMENT: "POSITION SHIFT" or "POSITION HELD" with reasoning
3. CONFIDENCE: [High/Medium/Low] with justification
4. KEY UNCERTAINTY: What would change your view
5. COGNITIVE MANDATE: ${mandate}

Be substantive. Credit good arguments from others.`;

        try {
          const response = await provider.call(respondentPrompt);
          return { name: provider.name, response };
        } catch (error) {
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

      emitThinkTankProgress({ sessionId, step: stepMap[roundNum], status: "complete" });

    } catch (error) {
      console.error(`Think Tank round ${roundNum} failed:`, error.message);
      break;
    }
  }

  emitThinkTankProgress({ sessionId, step: "summarising", status: "active" });

  await storage.updateBrainstormSession(sessionId, userId, {
    thinkTank: deliberationRounds,
    thinkTankStatus: "completed",
  });

  emitThinkTankProgress({ sessionId, step: "summarising", status: "complete" });
  emitThinkTankProgress({ sessionId, step: "complete", status: "complete" });
}
