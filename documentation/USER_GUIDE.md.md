# MicroXRCEForge Pro User Guide & Developer Manual

## 1\. Quickstart Guide

1. Launch `index.html` in your browser.  
2. In the "Entity & Session Config" panel, set your Client Session ID (e.g., `0x81`).  
3. Select stream mode (`Reliable Stream 0x80` or `Best Effort Stream 0x01`).  
4. Set the target ROS 2 / DDS topic name (e.g. `rt/turtle1/cmd_vel`).  
5. Click **Handshake Client** to emit a serialized `CREATE_CLIENT` submessage frame.  
6. Click **Publish (WRITE\_DATA)** to serialize your payload into CDR format and wrap it in an XRCE data frame.  
7. Click **Run Normative Audit** to evaluate all 10 protocol clauses against OMG DDS-XRCE v1.0.

## 2\. Standalone Headless Python CLI

Run automated regression testing and generate compliance reports:

python3 xrce-cli.py \--audit \--session 0x81 \--topic "rt/cmd\_vel"