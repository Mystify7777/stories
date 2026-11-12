# Chapter 6: The AI Service and the Deployment Seam

The AI service is intentionally small. FastAPI exposes a prediction endpoint, and the predictor lowercases input and counts occurrences of a fixed list of scam-associated keywords. Three or more matches return `MALICIOUS` with 0.9 confidence; two return `SUSPICIOUS` with 0.6; fewer return `SAFE` with 0.8.

This is a transparent heuristic, not a trained language model. The confidence values are constants associated with keyword-count bands, not empirically calibrated probabilities. A substring-based count can also match keywords inside larger words, and the approach has no semantic understanding of context, negation, or novel scam patterns. The implementation can be useful as a simple baseline, but its labels must not be presented as certainty.

## The service contract

The Express adapter posts `{ text }` to `${AI_SERVICE_URL}/api/predict` and returns the service response. The FastAPI route validates the body through a Pydantic model and calls the predictor. The two service boundaries are straightforward; deployment configuration is where a working local request can fail after deployment.

The backend environment example uses a local AI service address for development. In a split deployment, localhost means the backend's own runtime, not the separate AI service. The deployment guide says to configure `AI_SERVICE_URL` and verify backend-to-AI connectivity. The story should not claim that this is currently healthy without a live request from the deployed backend.

## What should be verified

1. Confirm the deployed backend's `AI_SERVICE_URL` points to the deployed FastAPI service, not localhost.
2. Call the AI service health/root endpoint and prediction endpoint independently.
3. Call `POST /api/ai/predict` through the deployed backend and verify the request reaches the expected AI host.
4. Exercise timeout, unavailable-service, invalid-input, and malformed-response behavior.
5. Review user-facing wording so a heuristic classification is not mistaken for a definitive safety verdict.

The backend's error path currently catches the adapter failure and returns a server error. This provides a failure response, but the public route and frontend still need integration verification to establish the complete user experience.

**Sources:** [Predictor](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/ai-service/app/services/predictor.py), [FastAPI route](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/ai-service/app/routes/predict.py), [Express AI adapter](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/services/aiService.js), [AI controller](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/controllers/aiController.js), [deployment guide](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/ops/DEPLOYMENT.md).
