# Moderator Code - Part 1: Think Tank Endpoint & Selection

## Part 1: Think Tank Endpoint & Moderator Selection

```typescript
// Think Tank deliberation - multi-round AI dialogue - protected
// Runs in BACKGROUND so user can navigate away without losing progress
app.post("/api/brainstorm/thinktank/:sessionId", isAuthenticated, async (req, res) => {
  const sessionId = parseInt(req.params.sessionId);
  const userId = (req.session as any).userId;
  const requestedModerator = req.body?.moderator;
  const session = await storage.getBrainstormSession(sessionId, userId);

  if (!session || !session.round1Responses) {
    return res.status(404).json({ error: "Session not found or Round 1 not completed" });
  }

  if (session.thinkTank && session.thinkTank.length > 0) {
    return res.json({ 
      sessionId, 
      status: "already_completed",
      deliberationRounds: session.thinkTank,
      message: "Think Tank already completed for this session"
    });
  }

  const providers = await getActiveProviders(userId, "thinktank");
  
  let moderator;
  const moderatorToUse = requestedModerator || session.selectedModerator;
  
  if (moderatorToUse) {
    moderator = providers.find(p => p.name === moderatorToUse);
    if (!moderator) {
      return res.status(400).json({ error: `Requested moderator "${moderatorToUse}" is not available` });
    }
  } else {
    const moderatorOrder = ["Claude", "ChatGPT", "Gemini", "DeepSeek", "Grok"];
    moderator = moderatorOrder.map(name => providers.find(p => p.name === name)).filter(Boolean)[0];
  }

  if (!moderator) {
    return res.status(500).json({ error: "No AI provider available for moderating" });
  }

  await storage.updateBrainstormSession(sessionId, userId, { thinkTankStatus: "running" });

  res.json({ 
    sessionId, 
    status: "started",
    moderator: moderator.name,
    message: "Think Tank started. You can safely navigate away."
  });

  runThinkTankInBackground(sessionId, session, providers, moderator, userId).catch(async (err) => {
    emitThinkTankProgress({ sessionId, step: "error", status: "error", error: err.message });
    await storage.updateBrainstormSession(sessionId, userId, { thinkTankStatus: "idle" });
  });
});
