# 🛡️ SDN-NetGuard: Real-Time DDoS Defense with ML & SDN

![Python](https://img.shields.io/badge/Python-3.10+-blue?style=for-the-badge&logo=python&logoColor=white)
![Java](https://img.shields.io/badge/Java-JDK_1.8-red?style=for-the-badge&logo=java&logoColor=white)
![Mininet](https://img.shields.io/badge/Mininet-SDN-black?style=for-the-badge&logo=linux&logoColor=white)
![Scikit-Learn](https://img.shields.io/badge/Scikit--Learn-Machine_Learning-F7931E?style=for-the-badge&logo=scikit-learn&logoColor=white)

--- 

## System Preview

Below is the overarching architecture of the real-time DDoS detection and mitigation pipeline.

<p align="center">
  <img src="imgs/Tong%20quan%20quy%20trinh.png" alt="System Workflow Architecture" width="850">
</p>

---

## I. The Backstory & Motivation

### Academic Context & Origin
This project was developed as the final term report for the **Network Security (Bảo mật mạng)** course.

It investigates the intersection of **Software-Defined Networking (SDN)** and **Machine Learning (ML)** to address one of the most persistent threats in cybersecurity: Distributed Denial of Service (DDoS) attacks. 

Traditional firewalls often rely on static rules or threshold-based blocking, which can be easily circumvented by sophisticated attacks or result in false positives that block legitimate traffic. The motivation behind this project was to build a closed-loop, automated defense mechanism where a centralized SDN controller acts as the "brain," dynamically programming the network based on the real-time analytical intelligence provided by a Machine Learning model.

---

## II. Project Overview

### Executive Summary
- **SDN-NetGuard** is a comprehensive, real-time DDoS detection and mitigation framework deployed on a simulated Mininet network.
- The system captures network traffic on-the-fly, extracts over 80 behavioral features using CICFlowMeter, and feeds them into a pre-trained ML model (Random Forest based on the CICIDS2017 dataset).
- Upon detecting malicious traffic patterns, the ML module communicates directly with the **Floodlight SDN Controller** via REST APIs to instantly inject OpenFlow `DROP` rules, isolating the attacker at the switch level without disrupting legitimate users.

### Tech Stack
The ecosystem integrates network simulation, packet analysis, and predictive modeling:

| Component | Technology | Role & Features |
| :--- | :--- | :--- |
| **SDN Controller** | `Floodlight (Java)` | Manages OpenFlow switches and provides a REST API to dynamically push flow rules. |
| **Network Simulation** | `Mininet` | Creates the virtual topology containing a Web Server (Victim), Normal Host, and Attacker Host. |
| **Packet Processing** | `tcpdump` & `CICFlowMeter` | Captures traffic in 15-second windows and extracts 80+ network flow features into CSV format. |
| **Machine Learning** | `Python (scikit-learn)` | Trains a RandomForest model and evaluates real-time CSV streams against a calibrated threshold. |

---

## 🚀 Getting Started (Local Lab)

To deploy this defense system locally, follow these steps. *Note: A Linux environment (Ubuntu 22.04+ VM) is strictly required.*

### 1. Prerequisites
Ensure you have Python 3.10+ and Java JDK 1.8 installed. Root (`sudo`) privileges are required for network operations.
- Extract the `.venv.zip` to `.venv` to load all Python dependencies.
- Ensure CICFlowMeter is built and placed in the project root.
- The Floodlight controller must be pre-built (`target/floodlight.jar`).

### 2. Phase 1: Controller & ML Initialization
Open two separate terminals in the `source/` directory.

**Terminal 1 (Floodlight Controller):**
```bash
sudo java -jar target/floodlight.jar
```
*Verify controller status at `http://localhost:8080/wm/core/controller/switches/json`.*

**Terminal 2 (Train the ML Model):**
```bash
python3 machinelearning/ML_trainer.py --csv dataset/CICIDS2017_processed.csv
```
*This generates `model.pkl`, `metadata.pkl`, and evaluation charts (ROC, PR-curve).*

### 3. Phase 2: Network Topology & Real-time Defense
Start the Mininet topology in a new terminal:
```bash
python3 mininet/topology.py
```
From the Mininet CLI, open xterm windows for the hosts (`mininet> xterm h1 h2 h3`).

**Launch the Automated Defense Pipeline (Run concurrently in new terminals):**
1. **Packet Capture:** `./processing/capture_tcpdump.sh` (Saves 15s PCAP windows).
2. **Feature Extraction:** `./processing/pcap_processor.sh` (Converts PCAPs to `Predict.csv`).
3. **ML Controller:** `python3 controller/realtime_floodlight_ML.py --csv output/final_csv/Predict.csv --threshold 0.07`

### 4. Trigger the Attack
On the **h3** xterm window, launch the DDoS script:
```bash
sh mininet/ddos.sh
```
**Expected Outcome:** The ML Controller terminal will log `Block src [Attacker_IP]`. The Floodlight controller will push a `DROP` rule, and traffic from `h3` to the web server (`h1`) will be severed within 10-20 seconds, while legitimate traffic from `h2` remains unaffected.

---

## III. Network Architecture & Traffic Flow

The defensive workflow operates as a continuous, automated feedback loop:

1. **Traffic Generation:** Mininet hosts generate mixed traffic (benign pings/HTTP requests and malicious floods).
2. **Real-Time Capture:** `tcpdump` sniffs packets directly from the OpenFlow switch interface.
3. **Feature Extraction:** `CICFlowMeter` parses the raw PCAPs and calculates complex statistical features (e.g., flow duration, packet length variance).
4. **ML Inference:** The `realtime_floodlight_ML.py` script ingests the CSV data, applies scaling, and uses the RandomForest model to predict the probability of a DDoS attack.
5. **SDN Mitigation:** If the probability exceeds the threshold (0.07), a REST API `POST` request is sent to the Floodlight controller.
6. **Rule Injection:** Floodlight pushes a high-priority OpenFlow `DROP` rule to the switch, blocking the specific attacker's IP.

---

## IV. Current Limitations

While this system successfully demonstrates AI-driven SDN security, it has experimental constraints:

| Limitation | Technical Reason | Potential Risk |
| :--- | :--- | :--- |
| **Processing Latency** | Capturing PCAPs (15s window) and parsing them via Java-based CICFlowMeter introduces a noticeable delay. | Fast-acting DDoS attacks (e.g., UDP amplification) might overwhelm the server before the ML model has time to react and push the block rule. |
| **External Dependency** | Relies on `tcpdump` and external PCAP processing rather than querying the SDN switch directly. | Increases CPU overhead and reduces the elegance of a pure SDN architecture. |
| **Model Drift** | The model is statically trained on the CICIDS2017 dataset. | Zero-day attack patterns or newly morphed DDoS signatures might evade detection. |

---

## V. Future Roadmap

To evolve this Proof of Concept into a production-ready intrusion prevention system, the following upgrades are planned:

| Phase | Upgrade Module | Technical Details | Core Objective |
| :---: | :--- | :--- | :--- |
| **Phase 2** | **Native SDN Polling** | Bypass `tcpdump` and `CICFlowMeter` by using Floodlight's REST API to periodically poll OpenFlow switch statistics (packet_in rates, byte counts). | Drastically reduce detection latency from ~15-20s down to < 2 seconds and lower CPU overhead. |
| **Phase 3** | **Deep Learning (DL)** | Replace RandomForest with sequence-based models like LSTM or 1D-CNNs. | Improve detection accuracy for complex, low-rate (Slowloris) application-layer DDoS attacks. |
| **Phase 4** | **Adaptive Thresholding**| Implement dynamic anomaly thresholds based on time-of-day traffic baselines rather than a hardcoded `0.07`. | Reduce false positive rates during legitimate traffic spikes (flash crowds). |

---

<p align="center">
  <b>Built for advancing the frontier of Software-Defined Security 🚀</b>
</p>
# 🛡️ SDN-NetGuard: Real-Time DDoS Defense with ML & SDN

![Python](https://img.shields.io/badge/Python-3.10+-blue?style=for-the-badge&logo=python&logoColor=white)
![Java](https://img.shields.io/badge/Java-JDK_1.8-red?style=for-the-badge&logo=java&logoColor=white)
![Mininet](https://img.shields.io/badge/Mininet-SDN-black?style=for-the-badge&logo=linux&logoColor=white)
![Scikit-Learn](https://img.shields.io/badge/Scikit--Learn-Machine_Learning-F7931E?style=for-the-badge&logo=scikit-learn&logoColor=white)

--- 

## System Preview

Below is the overarching architecture of the real-time DDoS detection and mitigation pipeline.

<p align="center">
  <img src="imgs/Tong%20quan%20quy%20trinh.png" alt="System Workflow Architecture" width="850">
</p>

---

## I. The Backstory & Motivation

### Academic Context & Origin
This project was developed as the final term report for the **Network Security (Bảo mật mạng)** course.

It investigates the intersection of **Software-Defined Networking (SDN)** and **Machine Learning (ML)** to address one of the most persistent threats in cybersecurity: Distributed Denial of Service (DDoS) attacks. 

Traditional firewalls often rely on static rules or threshold-based blocking, which can be easily circumvented by sophisticated attacks or result in false positives that block legitimate traffic. The motivation behind this project was to build a closed-loop, automated defense mechanism where a centralized SDN controller acts as the "brain," dynamically programming the network based on the real-time analytical intelligence provided by a Machine Learning model.

---

## II. Project Overview

### Executive Summary
- **SDN-NetGuard** is a comprehensive, real-time DDoS detection and mitigation framework deployed on a simulated Mininet network.
- The system captures network traffic on-the-fly, extracts over 80 behavioral features using CICFlowMeter, and feeds them into a pre-trained ML model (Random Forest based on the CICIDS2017 dataset).
- Upon detecting malicious traffic patterns, the ML module communicates directly with the **Floodlight SDN Controller** via REST APIs to instantly inject OpenFlow `DROP` rules, isolating the attacker at the switch level without disrupting legitimate users.

### Tech Stack
The ecosystem integrates network simulation, packet analysis, and predictive modeling:

| Component | Technology | Role & Features |
| :--- | :--- | :--- |
| **SDN Controller** | `Floodlight (Java)` | Manages OpenFlow switches and provides a REST API to dynamically push flow rules. |
| **Network Simulation** | `Mininet` | Creates the virtual topology containing a Web Server (Victim), Normal Host, and Attacker Host. |
| **Packet Processing** | `tcpdump` & `CICFlowMeter` | Captures traffic in 15-second windows and extracts 80+ network flow features into CSV format. |
| **Machine Learning** | `Python (scikit-learn)` | Trains a RandomForest model and evaluates real-time CSV streams against a calibrated threshold. |

---

## 🚀 Getting Started (Local Lab)

To deploy this defense system locally, follow these steps. *Note: A Linux environment (Ubuntu 22.04+ VM) is strictly required.*

### 1. Prerequisites
Ensure you have Python 3.10+ and Java JDK 1.8 installed. Root (`sudo`) privileges are required for network operations.
- Extract the `.venv.zip` to `.venv` to load all Python dependencies.
- Ensure CICFlowMeter is built and placed in the project root.
- The Floodlight controller must be pre-built (`target/floodlight.jar`).

### 2. Phase 1: Controller & ML Initialization
Open two separate terminals in the `source/` directory.

**Terminal 1 (Floodlight Controller):**
```bash
sudo java -jar target/floodlight.jar
```
*Verify controller status at `http://localhost:8080/wm/core/controller/switches/json`.*

**Terminal 2 (Train the ML Model):**
```bash
python3 machinelearning/ML_trainer.py --csv dataset/CICIDS2017_processed.csv
```
*This generates `model.pkl`, `metadata.pkl`, and evaluation charts (ROC, PR-curve).*

### 3. Phase 2: Network Topology & Real-time Defense
Start the Mininet topology in a new terminal:
```bash
python3 mininet/topology.py
```
From the Mininet CLI, open xterm windows for the hosts (`mininet> xterm h1 h2 h3`).

**Launch the Automated Defense Pipeline (Run concurrently in new terminals):**
1. **Packet Capture:** `./processing/capture_tcpdump.sh` (Saves 15s PCAP windows).
2. **Feature Extraction:** `./processing/pcap_processor.sh` (Converts PCAPs to `Predict.csv`).
3. **ML Controller:** `python3 controller/realtime_floodlight_ML.py --csv output/final_csv/Predict.csv --threshold 0.07`

### 4. Trigger the Attack
On the **h3** xterm window, launch the DDoS script:
```bash
sh mininet/ddos.sh
```
**Expected Outcome:** The ML Controller terminal will log `Block src [Attacker_IP]`. The Floodlight controller will push a `DROP` rule, and traffic from `h3` to the web server (`h1`) will be severed within 10-20 seconds, while legitimate traffic from `h2` remains unaffected.

---

## III. Network Architecture & Traffic Flow

The defensive workflow operates as a continuous, automated feedback loop:

1. **Traffic Generation:** Mininet hosts generate mixed traffic (benign pings/HTTP requests and malicious floods).
2. **Real-Time Capture:** `tcpdump` sniffs packets directly from the OpenFlow switch interface.
3. **Feature Extraction:** `CICFlowMeter` parses the raw PCAPs and calculates complex statistical features (e.g., flow duration, packet length variance).
4. **ML Inference:** The `realtime_floodlight_ML.py` script ingests the CSV data, applies scaling, and uses the RandomForest model to predict the probability of a DDoS attack.
5. **SDN Mitigation:** If the probability exceeds the threshold (0.07), a REST API `POST` request is sent to the Floodlight controller.
6. **Rule Injection:** Floodlight pushes a high-priority OpenFlow `DROP` rule to the switch, blocking the specific attacker's IP.

---

## IV. Current Limitations

While this system successfully demonstrates AI-driven SDN security, it has experimental constraints:

| Limitation | Technical Reason | Potential Risk |
| :--- | :--- | :--- |
| **Processing Latency** | Capturing PCAPs (15s window) and parsing them via Java-based CICFlowMeter introduces a noticeable delay. | Fast-acting DDoS attacks (e.g., UDP amplification) might overwhelm the server before the ML model has time to react and push the block rule. |
| **External Dependency** | Relies on `tcpdump` and external PCAP processing rather than querying the SDN switch directly. | Increases CPU overhead and reduces the elegance of a pure SDN architecture. |
| **Model Drift** | The model is statically trained on the CICIDS2017 dataset. | Zero-day attack patterns or newly morphed DDoS signatures might evade detection. |

---

## V. Future Roadmap

To evolve this Proof of Concept into a production-ready intrusion prevention system, the following upgrades are planned:

| Phase | Upgrade Module | Technical Details | Core Objective |
| :---: | :--- | :--- | :--- |
| **Phase 2** | **Native SDN Polling** | Bypass `tcpdump` and `CICFlowMeter` by using Floodlight's REST API to periodically poll OpenFlow switch statistics (packet_in rates, byte counts). | Drastically reduce detection latency from ~15-20s down to < 2 seconds and lower CPU overhead. |
| **Phase 3** | **Deep Learning (DL)** | Replace RandomForest with sequence-based models like LSTM or 1D-CNNs. | Improve detection accuracy for complex, low-rate (Slowloris) application-layer DDoS attacks. |
| **Phase 4** | **Adaptive Thresholding**| Implement dynamic anomaly thresholds based on time-of-day traffic baselines rather than a hardcoded `0.07`. | Reduce false positive rates during legitimate traffic spikes (flash crowds). |

---

<p align="center">
  <b>Built for advancing the frontier of Software-Defined Security 🚀</b>
</p>
