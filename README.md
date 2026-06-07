# Leleka100_troubleshooting
This report details an FCC connectivity failure in the Leleka-100 UAV GCS. Though telemetry displayed, the FCC couldn’t connect. Root cause: MAVLink data was piped directly to DFA, never reaching the network. A non-invasive MAVProxy forwarder resolved it. Methodology and solution included.
