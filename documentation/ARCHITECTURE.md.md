# MicroXRCEForge Pro Technical Architecture & Specification

## 1\. Protocol Architecture & Standards

OMG DDS-XRCE v1.0 (formal/19-05-02) standardizes communication between eXtremely Resource Constrained Environments (XRCE Clients, e.g. microcontrollers) and full DDS networks via an intermediary (XRCE Agent).

### Architectural Pillars:

1. **Client-Agent Model**: Microcontrollers maintain minimal state and zero discovery overhead; the full DDS Participant, Readers, and Writers are instantiated in the Agent on behalf of the Client.  
2. **4-Byte Frame Headers**: Header overhead is capped at 4 bytes: `[SessionId (1B), StreamId (1B), SequenceNumber (2B)]`.  
3. **Stream Reliability Modes**:  
   - Best-Effort Streams (1–127): Low latency, sequence unchecked.  
   - Reliable Streams (128–255): Windowed delivery confirmed via HEARTBEAT / ACKNACK submessages.  
4. **Common Data Representation (CDR)**: Endianness-flagged native binary serialization with strict 4-byte and 8-byte primitive alignments.  
5. **micro-ROS & PX4 Bridge Compatibility**: Direct support for FreeRTOS, Zephyr, and NuttX micro-ROS clients connecting to ROS 2 Humble/Iron/Jazzy graphs.