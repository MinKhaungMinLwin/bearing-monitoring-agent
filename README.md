# Bearing Monitoring Agent (AI Mastery Capstone, Project 6)

This project is a decision-support agent for the IMS bearing run-to-failure data. A simple rule-based router keeps an eye on each bearing's health indicators and only wakes the LLM agent (gpt-4o-mini, through OpenAI function calling) when something changes. The agent then looks at the data through read-only tools and its own memory, and recommends a status. Guardrails check every decision before it is accepted, and an escalation is only a request for a human to review it. The agent never acts on the rig itself.

## Repository
- `agentic_system.ipynb`: the whole pipeline, from features to evaluation, with my interpretation of each result.
- `results/`: the features for tests 2 and 3, the agent's decisions on both tests, the consistency runs and the off-scope answers (CSV and JSON).
- `logs/`: JSONL traces of the full runs on tests 2 and 3. They record every tool call, rejection, forced decision and final decision.
- `figures/`: Figures 1–4. `arch.dot` is the Graphviz source of Figure 4, the architecture diagram.
- `requirements.txt`: the exact package versions I used (Python 3.15).
- `.env.example`: a template for your own API settings.

## Data (not included)
The data is the IMS Bearing Data Set (Lee et al., 2007), provided by the Center for Intelligent Maintenance Systems, University of Cincinnati, and published in the NASA Prognostics Data Repository: https://data.nasa.gov/dataset/ims-bearings (direct ZIP: https://phm-datasets.s3.amazonaws.com/NASA/4.+Bearings.zip). Its Readme names Qiu et al. (2006) as the reference paper. Download it and put the files under `data/ims/`:

- `2nd_test.rar` unpacks to `data/ims/2nd_test/`.
- `3rd_test.rar` unpacks to `data/ims/4th_test/txt/`. The folder name really is `4th_test`.

Note: the dataset Readme says the third test has 4,448 files and ended on 4 April 2004, but the archive holds 6,324 files that run until 18 April. Section 15 of the notebook explains why this matters.

## Setup
1. `pip install -r requirements.txt`
2. Copy `.env.example` to `.env` and fill in your own key and base URL. **No API key is included in this repository.**

## Reproducing
The four expensive steps each have a flag: `RUN_AGENT` (full test 2 run, about 30 minutes), `RUN_CONSISTENCY`, `RUN_OFFSCOPE` and `RUN_AGENT3` (full test 3 run, about 100 minutes). All four are set to `False`, so those cells load the saved results from `results/` and `logs/` instead of calling the LLM again. Set a flag to `True` to rerun that part. The results will differ a little each time, because LLM outputs are not fully repeatable.

A Restart & Run All still needs two things:
- **An API key.** The setup cell creates the client and sends one test call, and the Trial 2 cell (section 9) makes about 4 agent assessments.
- **The raw data.** Sections 1 and 2 compute the test 2 features from the raw files, and section 15 reads the list of test 3 files. Test 3's features are cached in `results/test3_features.csv`.

## References
Lee, J., Qiu, H., Yu, G., Lin, J., & Rexnord Technical Services. (2007). *Bearing data set* [Data set]. IMS, University of Cincinnati; NASA Prognostics Data Repository, NASA Ames Research Center. https://data.nasa.gov/dataset/ims-bearings

Qiu, H., Lee, J., Lin, J., & Yu, G. (2006). Wavelet filter-based weak signature detection method and its application on rolling element bearing prognostics. *Journal of Sound and Vibration, 289*(4–5), 1066–1090. https://doi.org/10.1016/j.jsv.2005.03.007
