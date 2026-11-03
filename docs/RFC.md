# RFC: AI-Enabled Connected Digital Twin Platform for EV Battery Fleet Monitoring

## 1. Project Description

**Domain:** Energy & EV Battery Fleet Management

**What We Want to Solve:**  
EV batteries degrade over time, but individual cells often age unevenly. A single defective cell develops higher internal resistance, leading to rapid voltage drops and excessive localized heating during heavy acceleration or fast charging. A fleet operator cannot manually inspect hundreds of vehicles, while onboard Battery Management Systems (BMS) only react after hard safety limits are crossed, often shutting the vehicle down abruptly. Surface temperature sensors also react with delay, and existing fleet platforms merely log data passively without taking proactive steps. Operators need to know which batteries are abnormal (without false alarms), understand their root cause, and apply timely mitigations before cars break down.

**Our Idea:**  
We want to develop an AI-enabled connected digital twin platform that continuously monitors cell-level health across a simulated EV fleet and automatically mitigates stress before hard safety cutoffs. The platform closes the control loop by feeding interventions directly back into the running simulation, such as throttling current to let a struggling cell cool down safely while a human operator can inspect diagnostics and ask questions via an agent.

**Data Sources:**  
The system's software simulator is calibrated using two complementary datasets:
* **Fleet Telemetry Data:** IVST Changan EV Dataset ([Nature Communications 2025 cohort](https://doi.org/10.1038/s41467-025-56485-7)), providing multi-cell series readings from 300 commercial vehicles in real traffic.
* **Laboratory Aging Baseline:** [Oxford Battery Degradation Dataset](https://ora.ox.ac.uk/objects/uuid:03ba4b01-cfed-46d3-9b1a-7d4a7bdf6fac), providing controlled degradation curves and reference capacity metrics.

**Asynchronous Streams:**  
The simulator emits two independent, decoupled streams:
* **Battery Telemetry Stream:** Periodic CAN frames sent every 10 seconds, containing cell voltages, pack current, and temperature probes.
* **Operations & Environment Stream:** Asynchronous events carrying charging states at the depot, vehicle speed, dispatch priority, and ambient weather.

---

## 2. Problem Definition

Building this system raises four technical challenges:

1. **Two data streams that do not arrive together.**  
   Battery readings arrive every 10 seconds, but events like charging or weather come at random times, late, or out of order. We have to match each event to the right vehicle and the right second without slowing down live monitoring.

2. **Telling a weak cell from hard acceleration.**  
   When a driver accelerates hard, every cell’s voltage drops a little, and 10-second sampling smooths out these rapid spikes. We have to tell this normal drop apart from a cell that is actually dying, especially since real field data comes with zero fault labels.

3. **Keeping AI agent decisions safe.**  
   An LLM agent can hallucinate dangerous actions, pick wrong parameters, or fail to respond within the 10-second telemetry window. We must limit it to a strict whitelist of verified control tools and ensure hardcoded BMS fallback rules trigger immediately if the agent times out or crashes.

4. **Proving the reliability of the predictions**  
   Real driving logs never mark which cell is dying, so we cannot just check answers against an answer key. To prove our model actually works, we test it on cars it has never seen: we compare its 10-minute temperature forecasts against what those cars actually measured, and we inject fake cell faults into normal trips to see if it catches the problem in time without spamming false warnings.

---

## 3. State of the Art


1. **Fleet telematics ([Geotab](https://www.geotab.com/fleet-management-solutions/electric-vehicles/), [ViriCiti](https://www.chargepoint.com/about/news/chargepoint-acquires-ebus-and-commercial-vehicle-management-provider-viriciti-accelerate/)).**  
   *What it does:* collects vehicle data in the cloud and shows battery charge, health and charging 
   *Pros:* scales to thousands of vehicles and adds location and charging context.  
   *Cons:* mainly monitoring and reporting; to our knowledge no automatic actions back to the vehicle.

2. **Battery digital twins ([TWAICE](https://www.twaice.com/)).**  
   *What it does:* cloud models of each battery, updated from field data, to track ageing and remaining life.  
   *Pros:* supports maintenance planning, health certificates and warranty claims.  
   *Cons:* built for slow health reports, not real-time decisions, as far as public material shows.

3. **Research on real EV data (He et al., [EVBattery](https://arxiv.org/abs/2201.12358) ([dataset](https://figshare.com/articles/dataset/EVBattery_A_Large-Scale_Electric_Vehicle_Dataset_for_Battery_Health_and_Capacity_Estimation/23301881)), 2022; [Liu et al., 2025](https://doi.org/10.1038/s41467-025-56485-7)).**  
   *What it does:* machine learning for health estimation and anomaly detection.  
   *Pros:* real data and open benchmarks.  
   *Cons:* offline and open-loop; EVBattery gives only average cell voltage; none connects the model to an agent that tests mitigations.

**Gap:** None combines cell-level weak-cell detection, a temperature forecast and an AI agent that acts safely and feeds back into the system. We do this in simulation.

---

## 4. Proposed Solution

* **Simulator.** A configurable simulator creates a fleet of virtual electric vehicles. Its settings (fleet size, driving profiles, seasons, fault types and timing) live in one configuration file, and a fixed random seed makes every run reproducible. Driving profiles come from real fleet data. A software battery model, an equivalent-circuit model calibrated on fleet and laboratory ageing data, calculates how each cell’s voltage and temperature respond to the current. Faults are injected by making one cell age faster than the others.

* **Two independent streams.** The first stream is battery telemetry: cell voltages, pack current and temperatures every 10 seconds. The second carries operations and environment events: charging sessions, route assignments and weather. They use separate clocks with random delays, so events arrive late or out of order, as in a real fleet.

* **Ingestion and storage.** A backend service checks each record for valid ranges and format, marks missing or broken values instead of guessing them, aligns the two streams per vehicle by timestamp, and stores the clean data in a database. Stored data supports replay of past incidents and offline training.

* **Online and offline processing.** Online, the system computes features over a sliding time window and runs the models as data arrives. Offline, a scheduled job builds training and test sets from the stored history and replays selected incidents to test changes.

* **Two models.** The weak-cell detector estimates each cell’s resistance by comparing its voltage drop with the pack median under the same current, which cancels the dip that hard acceleration causes in every cell, and it flags outliers with an unsupervised model. The temperature forecaster predicts each cell’s temperature 10 minutes ahead from the last 15 minutes of current, voltage and ambient temperature.

* **Routing and decision agent.** To keep cost and delay low, simple checks run on every reading, the models run only on suspicious windows, and the agent is called only when risk is high. The agent is a language model connected through an MCP server that gives it controlled access to cell data, forecasts and trip context. It must choose from a short list of approved actions: cap the current by 20% or 30%, reschedule charging, or open a maintenance ticket. If its answer is invalid or later than 10 seconds, fixed safety rules take over.

* **Feedback loop.** The chosen action is sent back to the simulator, which recalculates voltage and heat for the rest of the trip. This shows whether the action kept the cell safe while the vehicle still finished its route.

* **Interface.** A dashboard shows fleet status, cell-level detail, and the agent’s decisions with short explanations.

* **Evaluation.** Model level: temperature forecast error, and precision and recall of fault detection on vehicles and fault types held out from training. End to end: peak cell temperature and minutes of reduced power, compared with a run without the agent. Stress scenarios: lost data, a frozen sensor, a seasonal temperature change, and a load spike with many more vehicles.