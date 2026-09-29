# Two Timer Modes: `global` and `strict`

A real CPNS/BUMN/OJK tryout uses one global countdown with free movement between Questions, but the original request for this platform was a budget per Question. We support both: `global` (one countdown for the whole Attempt) and `strict` (a budget per Question = total time ÷ number of Questions, the leftover budget burns when the Participant presses Next early, and there is no way back).

Reasons: one mode alone would force one kind of exam into a bad fit — `global` loses the pressure of a per-Question clock, `strict` breaks the CAT habit. The consequences: the session engine has two deadline paths, the Attempt deadline and the running Question deadline, and the server computes both, so closing the browser stops nothing. In either mode a Participant may submit early.
