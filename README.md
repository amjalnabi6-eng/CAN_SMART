
Why this project exists
Modern vehicles run on CAN, a serial bus designed in 1986 for reliability rather than security. CAN frames carry no authentication, no encryption, and no sender identity — any node on the bus can broadcast any ID, and every other node will accept it. This has led to public demonstrations of remote vehicle compromise (Miller & Valasek's 2015 Jeep Cherokee hack being the most famous).
Most existing CAN IDS solutions are either:

Closed-source commercial products (Karamba, GuardKnox, Argus) that researchers cannot inspect or reproduce, or
Heavy machine-learning models that need large labelled attack datasets and produce unexplainable alerts.

This project takes the opposite approach: a ~150-line Python tool with three transparent statistical detectors that any researcher or student can run on any candump log without GPUs, training data, or hardware dependencies.
Features

Three complementary detectors, each targeting a specific attack class
Pure statistics, no ML — every threshold is a visible constant, every alert traces to a single rule
Zero training data required — runs on day one against any candump log
Batch mode — point it at a directory and get one report per file plus a summary
Plain-text reports labelled by attack type — grep-able, version-controllable, printable
~150 lines of Python, dependencies: pandas only

How it works
The three detection methods
MethodRuleCatchesSEC-01 — FrequencyAn ID's frame count exceeds mean + 3σ of the distributionDoS / flooding attacks where one ID dominates the busSEC-02 — Payload
variationAn ID has more than 50 distinct payloadsInjection / fuzzing where an attacker sends randomized payloadsSEC-03 — State machineA monitored safety-critical ID exceeds 
its expected state countSpoofing / replay where forged frames introduce invalid states
All thresholds are named constants at the top of intrusion_scan.py:
pythonFREQ_SIGMA_MULTIPLIER = 3                      # SEC-01 sensitivity
PAYLOAD_VARIATION_LIMIT = 50                   # SEC-02 sensitivity
CRITICAL_IDS = {'0C1': 4, '0C5': 4}            # SEC-03 watchlist (ID → max valid states)
Installation
Requires Python 3.8 or newer.
bashgit clone https://github.com/<your-username>/can-bus-intrusion-detection.git
cd can-bus-intrusion-detection
pip install pandas
Usage
Single directory of logs
Edit the bottom of intrusion_scan.py to point at your log directory, then run it:
pythonTARGET_DIR = "/path/to/your/can/logs/"
bashpython3 intrusion_scan.py
Every .log file in the directory will be analyzed, and a report named intrusion_report_<logname>.txt will be written alongside each one.
Capturing your own logs
If you have access to a vehicle's CAN bus (or a CAN simulator), use Linux's can-utils:
bashsudo apt install can-utils
candump -L can0 > my_capture.log
The -L flag produces exactly the format this tool expects: (timestamp) interface ID#payload.
Example output
Running the detector on a 215,669-frame log with three injected attacks:
=== CAN BUS INTRUSION DETECTION REPORT ===

[SEC-01] FREQUENCY ANALYSIS
Attack Type : Denial of Service / Flooding
Threshold   : mean + 3σ (12380 msgs)
 ALERT: ID 7FF - 20000 msgs (247.5 msg/sec)

[SEC-02] PAYLOAD VARIATION ANALYSIS
Attack Type : Payload Injection / Fuzzing
Threshold   : more than 50 unique payloads per ID
 ALERT: ID 3A0 has 8500 unique payloads.
 ALERT: ID 2A0 has 800 unique payloads.
 ALERT: ID 0F1 has 644 unique payloads.

[SEC-03] CRITICAL ID STATE CHECK
Attack Type : Spoofing / Replay
Monitoring  : 0C1, 0C5
 WARNING: ID 0C1 state machine disrupted (7 states, expected <= 4).
 OK: ID 0C5 stable with 4 states.
The detector caught all three injected attacks (7FF, 2A0, 0C1) by the correct method.
Validation
The project includes augment_log.py, which takes a clean baseline capture and injects three documented attacks plus a false-positive trigger:
Test caseIDExpectedResultDoS flood (20,000 frames in 5s)7FFSEC-01 alert✅ CaughtInjection (800 random payloads)2A0SEC-02 alert✅ Caught (exact count)Spoofing (3 forged states)0C1SEC-03 warning✅ Caught (7 states detected)Legitimate sensor (FP defence)3A0No SEC-01 alert✅ Passed (3σ holds)
Threshold tuning
The detector originally used mean + 2σ for SEC-01, which is the textbook default. On real vehicle traffic this generated 5 false positives because legitimate ECUs cycle at vastly different rates (10 ms vs 1 s), inflating standard deviation. Tightening to mean + 3σ:
MetricBefore (2σ)After (3σ)Threshold~6,658 msgs12,380 msgsFalse positives on baseline50DoS attack detectionYesYes (still caught at 20,000)Sensor false-positive defenceFailedPassed
The takeaway: threshold choice matters more than method choice. A simple rule tuned well outperforms a sophisticated rule tuned carelessly.
Limitations & honest caveats

Offline analysis only. The current pipeline reads .log files. Real-time alerting on a live candump stream is future work.
No ground-truth recall measurement. Validation uses controlled attack injection, not a labelled real-world attack dataset (those are scarce, vehicle-specific, and often classified).
Critical-ID list is hand-curated. SEC-03 monitors 0C1 and 0C5 based on prior knowledge. Scaling to a full DBC requires automated extraction.
SEC-02 cannot distinguish a fuzzer from a high-variance sensor. A legitimate temperature sensor producing thousands of unique values looks identical to a fuzzing attack at the variance level. This is a known statistical limitation, not a bug.

Future work

Automated baseline learning — record a clean reference capture, store per-ID statistics, and flag deviations from each ID's own historical pattern (instead of a global threshold).
Live-stream mode — wrap the parser around candump stdin for real-time alerting.
DBC-driven SEC-03 — automatically build the critical-ID state table from a vehicle's DBC file.
Entropy-based payload analysis — distinguish random fuzzing payloads (high entropy) from sensor readings (locally correlated) to remove the SEC-02 false positive.


Acknowledgements
Project developed at King Fahd University of Petroleum & Minerals, Department of Information & Computer Science, as part of the MX Project (2026).

Author: Abdullah Alabbdrab Alnabi
Mentor: Dr. Waleed Al-gobi
