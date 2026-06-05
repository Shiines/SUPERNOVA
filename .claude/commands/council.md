# /council [decision] — Adversarial Stress Test

## What It Does
5 agents debate a decision from opposing angles before you commit to it.

## Steps
1. Take the decision as input
2. Assign adversarial roles to 5 agents:
   - Optimist (best case)
   - Pessimist (worst case)
   - Devil's Advocate (what everyone's missing)
   - Financial (cost/ROI lens)
   - Client (client's perspective)
3. Each agent gives 2-3 lines max
4. Synthesis: recommendation + confidence level

## Output
```
/council — [decision]
━━━━━━━━━━━━━━━━━
OPTIMIST:          [view]
PESSIMIST:         [view]
DEVIL'S ADVOCATE:  [view]
FINANCIAL:         [view]
CLIENT:            [view]
━━━━━━━━━━━━━━━━━
VERDICT: [recommendation]
CONFIDENCE: [low / medium / high]
```
