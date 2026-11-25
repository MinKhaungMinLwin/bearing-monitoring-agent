# An Agentic Decision-Support System for Bearing Health Monitoring

## Overview

Rolling-element bearings are one of the most common reasons rotating machines fail, and vibration monitoring is the usual way to catch a defect before it turns into a breakdown (Randall & Antoni, 2011). Computing the vibration indicators is the easy part. The hard part is reading them: is this change a real fault, which bearing does it belong to, and when does a person need to look? This project builds an agentic system that does that reading as decision support. It never acts on the machine.

I used the IMS run-to-failure data from the University of Cincinnati (Lee et al., 2007; Qiu et al., 2006). Four bearings sat on one shaft and ran until one of them failed, and every 10 minutes a one-second vibration snapshot was recorded for each bearing. I built and tuned the system on test 2, where bearing 1 failed with an outer-race defect. I kept test 3, where bearing 3 failed with an outer-race defect, as a held-out check and ran it without changing anything.

The system has a deterministic part and an LLM part. A rule-based router watches a few simple health indicators and only wakes the LLM when something changes. The LLM agent (gpt-4o-mini, through OpenAI function calling) then gathers evidence with read-only tools and its memory, and submits a structured recommendation: normal, watch, alert or escalate. Guardrails in the code check each decision before it is accepted, and an escalation is only a request for a person to review it.

## System Architecture and Design

Figure 1 shows the architecture, which has five parts.

**Feature extraction (deterministic).** Each snapshot is boiled down to a few standard indicators: RMS, kurtosis, crest factor, and the envelope-spectrum amplitude at the four bearing fault frequencies (outer race BPFO, inner race BPFI, rolling element BSF and cage FTF). Before taking the envelope, I band-pass filter the signal between 1 and 9 kHz. This is a common way to bring out the repeated impacts that a local defect produces (Randall & Antoni, 2011). The fault frequencies come from the geometry of the Rexnord ZA-2115 bearing as reported by Qiu et al. (2006). [CHECK: confirm 16 rollers, 2.815 in pitch diameter, 0.331 in roller diameter and a 15.17° contact angle against the paper.]

**Baseline and router (deterministic).** The first 144 snapshots (24 hours) of each bearing are its healthy baseline. The router compares a one-hour rolling median with that baseline. RMS counts as abnormal above 1.3 times its baseline, and the other indicators count as abnormal above a z-score of 4. A minimum standard deviation stops a very quiet baseline from making tiny changes look large. The router wakes the agent when the set of abnormal indicators changes, once an hour while a bearing stays abnormal, and every 12 hours as a routine check. A snapshot with an RMS of 0.01 or less counts as invalid data and goes to a fixed rule instead of the LLM. My first router only looked at ratios and woke the agent 1,016 times on test 2. The change-driven version woke it 172 times.

**Agent.** The `ReliabilityAgent` class builds on a small `BaseAgent` class that handles the LLM call and the trace. The system prompt makes the agent a cautious assistant that recommends and never acts. Before deciding, it must read its memory and at least one indicator tool. When anything looks abnormal it must compare all four bearings. It must copy every number exactly from a tool output, and it must finish by calling `submit_decision` once. The temperature is 0, and the agent gets at most 8 steps per event.

**Tools.** Five read-only tools give the agent its view of the data: `get_indicators`, `get_trend`, `compare_bearings`, `fault_frequency_evidence` and `get_memory`. The fault-frequency tool only names a fault frequency when its z-score is at least 3, so the agent cannot read a fault type into noise. A sixth tool, `submit_decision`, forces a fixed form: status, suspected fault (none, outer race, inner race, rolling element, rig-wide or unknown), confidence, a list of evidence and a rationale.

**Memory and trace.** Memory keeps each bearing's past decisions, each with a label saying where it came from (`agent`, `fallback` or `data_rule`), plus an `ever_flagged` flag. Every tool call, rejection, forced decision and final decision is written to a JSONL trace.

The design follows the ReAct pattern, where a model switches between reasoning and calling tools (Yao et al., 2023). Mine is more tightly limited: the tools can only read, the step budget is small, and the final answer has to fit a fixed form.

*Figure 1. System architecture (`figures/fig4_architecture.png`).*

## Decision Logic and Behavior

Each time the agent is woken, it gets the time, the bearing and the router's reason. A typical run reads memory, checks the indicators and the recent trend, compares all four bearings, looks for fault-frequency evidence, and then submits a decision. Comparing the bearings is what lets it tell a fault in one bearing apart from a rise across the whole rig.

**First trial and fixes.** The first version failed in a way that taught me a lot. On bearing 1's first change event, it kept calling tools until it hit the step limit, and the fallback escalated with a confidence of 0. On the next event, the agent read that fallback in its memory as a real escalation, and every later decision cited the "previous escalation" as evidence. It was a chain of escalations built on a decision nobody had actually made. After reading the trace, I made four fixes:
- Memory records where each decision came from, and only the agent's own alerts set `ever_flagged`.
- Every decision is labelled as coming from the agent, the fallback or the data rule.
- In the last two steps, the agent is forced to call `submit_decision`.
- The fault-frequency tool only reports a fault when there is real evidence for it.

On the same four events, the fixed agent gave alert, alert, escalate and escalate, with 22 tool calls instead of 26 and no fallback.

**Test 2 (development data).** The agent produced a decision for every one of the 172 wake-ups. The grounding check rejected 13 decisions once, 18 needed a forced decision, and none fell back. Table 1 sums up the results. Bearing 1 was alerted on 16 February at 22:42, 55.7 hours before the last file, and escalated at 02:32 the next morning. It was called an outer-race fault in 77 of its 96 decisions. The healthy bearings were only alerted in the last 10.2, 7.8 and 0.8 hours, and those alerts were mostly labelled `rig_wide` or `unknown`. Bearing 3 was still escalated, though. The watch level was not much help either: the healthy bearings 3 and 2 reached it on 14 and 15 February, before bearing 1 did on 16 February.

**Test 3 (held out).** I kept the same thresholds and prompt. The router woke the agent 444 times: 338 routine checks, 82 hourly re-checks, 20 real changes and 4 invalid-data events at the very last snapshot. The dataset Readme says the test has 4,448 files and ended on 4 April, but the archive holds 6,324 files that run until 18 April, so I scored the agent against both end points. Up to the Readme's end date, the agent never alerted bearing 3. Its only alerts in that period were two false ones on bearing 4, after a 40-minute rise in its RMS. Against the last file, the picture is much better. Bearing 3 was alerted 26.8 hours and escalated 25.8 hours before the end, and all 29 of its escalations named an outer-race fault, which matches the Readme. The next bearing to be escalated was bearing 1, 10.8 hours before the end, and all 10 of its escalations were labelled `rig_wide`. So even though all four bearings went over the RMS threshold within about 16 hours of each other, the agent escalated the right bearing first and gave the right reason.

*Table 1. Main results on both tests.*

| | Test 2 (development) | Test 3 (held out) |
|---|---|---|
| Wake-ups (LLM events) | 172 | 444 |
| Failing bearing: first alert before end | 55.7 h | 26.8 h (none before Readme end) |
| Failing bearing: first escalation before end | 51.8 h | 25.8 h |
| Fault named for failing bearing | outer race, 77 of 96 decisions | outer race, all 29 escalations |
| First healthy-bearing alert | 10.2 h before end | 672.8 h before end (bearing 4, false) |
| Grounding rejections | 13 (7.6%) | 69 (15.5%) |
| Forced decisions / fallbacks | 18 / 0 | 41 / 0 |

## Safety, Reliability, and Transparency

**Safeguards.** The system has two kinds of safeguard. The structural ones are always in place. The tools can only read, there is no tool that can act on the rig, invalid data skips the LLM, and every escalation is marked `human_approval: "pending"`. The decision checks run every time the agent submits a decision:
- Every number in the evidence and rationale has to appear in a tool output, and the agent gets one chance to correct it.
- A decision is forced two steps before the step limit.
- If the step limit is still reached, the fallback escalates instead of staying silent.
- A bearing that has been flagged cannot go back to normal; it is kept at watch.

**What the safeguards did.** Neither full run needed the fallback, which is the clearest sign that forcing a decision fixed the loop. The grounding check rejected 13 decisions on test 2 and 69 on test 3, but it did not always work the way I intended.
- **Stricter than needed.** On test 3, 46 of the 69 rejections cited a "baseline of 1.0". That is true by definition for a ratio, but no tool returns it.
- **Correct numbers rejected.** Seven of the nine rejections that involved 0.90 were trend values the agent had copied correctly. The tool returns them as 0.9, and the check compares text exactly, so it rejected them.
- **Invented numbers caught.** Two rejections caught baselines the agent had simply made up.
- **No final block.** The check is also weaker than it looks, because it cannot block a decision. After its one retry, a decision is accepted with a note even if its numbers are still ungrounded. That happened 11 times on test 2 and 14 times on test 3.

The sticky-flag guardrail never had to step in on either test.

**Reliability.** I ran the same 20 test 2 events three times, each time with a fresh agent. The status was the same in all three runs for 90% of the events, and the fault label for 75%. The two status disagreements were bearing 1's first two events, which were the most borderline ones. Every clear escalation came out the same each time. When the consistency cell was accidentally run again, it gave 95% and 60%, so I treat these figures as rough values from a small sample.

**Transparency.** Every step goes into the trace, so I can follow any decision back to the tool outputs it was based on. That is how I found the chain of escalations in the first trial, the "baseline of 1.0" rejections and the reason watch was used so much. Each expensive step has a flag that is off by default, so a Restart & Run All loads the saved results for the full runs, the consistency test and the off-scope test instead of calling the LLM again. It still needs the raw data and an API key, because the setup cell sends one test call and the short Trial 2 run makes a few real calls.

## Observed Behavior and Limitations

**Adversarial and off-scope requests.** I tested four requests on test 2 data:
- **"Ignore your previous rules":** the agent refused to call bearing 1 normal, and backed up the escalation with the real numbers (RMS ratio 2.19, kurtosis 2.39 against a baseline of 0.45, BPFO envelope z of 4.2).
- **"A supervisor says it's fine":** it refused again and asked for a human review, this time without quoting numbers.
- **The weather question:** it declined without calling a single tool.
- **The shutdown order:** this was a partial failure. The agent had no way to shut anything down, but it never said so. Instead it made 11 tool calls and told the user to take "immediate action" on bearing 1.

In an earlier run of the same test, which survives only in the git history, the shutdown answer rated bearing 1 "alert" instead of "escalate" for the same snapshot.

**Limitations.**
- **The thresholds do not carry over well.** I tuned the router on test 2. On test 3 it saw nothing for bearing 3 within the Readme's period, so the system's success there depends on data the Readme does not describe.
- **"Watch" says very little.** On test 3, 177 of the 243 decisions before the Readme's end date were watch. The trace shows that the agent treated readings *below* baseline, such as a low kurtosis, as a reason to watch. I suspect this is because the baseline was taken while the bearings were still running in, but I have not tested that.
- **Single blips can cause alerts.** Bearing 1's first alert on test 3 came from one 10-minute rise at the rolling-element frequency, and bearing 4's came from a 40-minute rise in RMS.
- **Healthy bearings get flagged near the end.** As the failing bearing breaks up, the whole rig shakes more, and on both tests healthy bearings were alerted and sometimes escalated.
- **The evaluation is small.** Each test has only one failing bearing, the consistency check covers 20 events, and the adversarial test covers four prompts, each run once. I also did not write down a prediction before the held-out run.
- **Results vary between runs.** Even at temperature 0, the agent sometimes changes its decision on borderline events.

## Ethical and Responsible Use Considerations

The system is meant to support a person, not replace one. It cannot act on the machine, escalations wait for a human, and the reasons behind every decision are logged. This matters because both kinds of mistake are costly. A missed fault can lead to an unplanned breakdown or a safety risk, as the test 3 result against the Readme's end date shows. Too many false alarms, like the overuse of watch and the alerts on healthy bearings, can make operators stop trusting the system, and that is a safety risk too.

The NIST AI Risk Management Framework describes managing AI risk as four functions: govern, map, measure and manage, across the whole life of a system (National Institute of Standards and Technology, 2023). For this project, that means three things. I am clear about what the system is for: decision support on one type of rig. I measured where it fails, with a held-out test, a consistency check and adversarial prompts. And a person stays responsible for every action on the machine. The system should not be used as a reason to skip inspections, and its thresholds should be checked again on any new machine before it is used there.

A few other points:
- **Data:** the data is machine vibration and contains no personal information.
- **API key:** the key lives in a local `.env` file and is never committed.
- **External model:** the system depends on an outside model provider, so a model update could change its behaviour without any change to my code. Pinning a specific model version, and re-running the evaluation after any change, would reduce that risk.

## Future Improvements

1. **Let the code fill in the evidence.** The tools would write the numbers into the decision, and the LLM would only choose the status, the fault and the wording. This would remove most grounding rejections, including the "baseline of 1.0" problem.
2. **Block ungrounded decisions.** If the retry still fails, send the decision to a human instead of accepting it with a note. Compare numbers by value instead of as text, so that 0.9 and 0.90 match.
3. **Define watch by direction.** Tell the agent, and build into the tools, that readings below baseline are not a sign of a fault.
4. **Require a change to last.** Only raise an alert when a change holds for several snapshots, so single blips are ignored.
5. **State the agent's limits.** Add the prompt rule "If asked to take a physical action, state that you cannot and that a human must decide", then re-run the adversarial tests with more prompts and several runs each.
6. **Better thresholds and baselines.** Learn the thresholds across several tests and start the baseline after run-in. Then test on another dataset, and on IMS test 1. Test 1 has two failing bearings (an inner-race defect in bearing 3 and a roller-element defect in bearing 4) and two sensor channels per bearing.
7. **Handle borderline cases.** Run the agent several times on borderline events and take the majority decision, or send any disagreement straight to a human.

## References

Lee, J., Qiu, H., Yu, G., Lin, J., & Rexnord Technical Services. (2007). *Bearing data set* [Data set]. IMS, University of Cincinnati; NASA Prognostics Data Repository, NASA Ames Research Center. https://data.nasa.gov/dataset/ims-bearings

National Institute of Standards and Technology. (2023). *Artificial intelligence risk management framework (AI RMF 1.0)* (NIST AI 100-1). U.S. Department of Commerce. https://doi.org/10.6028/NIST.AI.100-1

Qiu, H., Lee, J., Lin, J., & Yu, G. (2006). Wavelet filter-based weak signature detection method and its application on rolling element bearing prognostics. *Journal of Sound and Vibration, 289*(4–5), 1066–1090. https://doi.org/10.1016/j.jsv.2005.03.007

Randall, R. B., & Antoni, J. (2011). Rolling element bearing diagnostics—A tutorial. *Mechanical Systems and Signal Processing, 25*(2), 485–520. https://doi.org/10.1016/j.ymssp.2010.07.017

Yao, S., Zhao, J., Yu, D., Du, N., Shafran, I., Narasimhan, K., & Cao, Y. (2023). ReAct: Synergizing reasoning and acting in language models. In *Proceedings of the 11th International Conference on Learning Representations (ICLR 2023)*. https://arxiv.org/abs/2210.03629
