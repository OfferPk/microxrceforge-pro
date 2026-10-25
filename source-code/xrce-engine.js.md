/\*\*  
 \* OMG DDS-XRCE v1.0 Core Wire & CDR Serialization Engine  
 \* Implements message framing, submessage parsing, CDR encoding,  
 \* object representation, and reliable stream state management.  
 \*/

// OMG DDS-XRCE Submessage Identifiers (Section 8.2)  
const XRCE\_SUBMSG\_TYPES \= {  
  CREATE\_CLIENT: 0x00,  
  CREATE: 0x01,  
  GET\_INFO: 0x02,  
  DELETE: 0x03,  
  STATUS\_AGENT: 0x04,  
  STATUS: 0x05,  
  INFO: 0x06,  
  WRITE\_DATA: 0x07,  
  READ\_DATA: 0x08,  
  DATA: 0x09,  
  ACKNACK: 0x0a,  
  HEARTBEAT: 0x0b,  
  RESET: 0x0c,  
  FRAGMENT: 0x0d  
};

// Object Kinds (Section 8.3)  
const XRCE\_OBJECT\_KINDS \= {  
  PARTICIPANT: 0x01,  
  TOPIC: 0x02,  
  PUBLISHER: 0x03,  
  SUBSCRIBER: 0x04,  
  DATAWRITER: 0x05,  
  DATAREADER: 0x06  
};

// Stream IDs  
const XRCE\_STREAM\_IDS \= {  
  NONE: 0x00,  
  BEST\_EFFORT\_MIN: 0x01,  
  BEST\_EFFORT\_MAX: 0x7f,  
  RELIABLE\_MIN: 0x80,  
  RELIABLE\_MAX: 0xff  
};

class MicroXRCEEngine {  
  constructor(sessionId \= 0x81, clientKey \= \[0x01, 0x02, 0x03, 0x04\]) {  
    this.sessionId \= sessionId;  
    this.clientKey \= clientKey;  
    this.streamSequences \= new Map(); // streamId \-\> seqNum  
  }

  getNextSequence(streamId) {  
    const cur \= this.streamSequences.get(streamId) \|\| 0;  
    const next \= (cur \+ 1\) & 0xffff;  
    this.streamSequences.set(streamId, next);  
    return cur;  
  }

  /\*\*  
   \* Encodes a 4-byte XRCE Header (SessionId, StreamId, SequenceNumber)  
   \*/  
  serializeHeader(streamId \= 0x80) {  
    const seq \= this.getNextSequence(streamId);  
    return {  
      bytes: \[  
        this.sessionId,  
        streamId,  
        seq & 0xff,  
        (seq \>\> 8\) & 0xff  
      \],  
      sessionId: this.sessionId,  
      streamId: streamId,  
      sequence: seq  
    };  
  }

  /\*\*  
   \* Serializes a 4-byte Submessage Header (SubmsgId, Flags, SubmsgLength)  
   \*/  
  serializeSubmessageHeader(submsgId, flags, payloadLength) {  
    return \[  
      submsgId,  
      flags,  
      payloadLength & 0xff,  
      (payloadLength \>\> 8\) & 0xff  
    \];  
  }

  /\*\*  
   \* Constructs CREATE\_CLIENT submessage frame  
   \*/  
  serializeCreateClient(clientKey \= this.clientKey, session \= this.sessionId) {  
    const header \= this.serializeHeader(0x00); // Session management uses stream 0  
    const payload \= \[  
      ...clientKey,  
      session,  
      0x01, // Format: XRCE v1.0  
      0x00, // Reserved  
      0x00  
    \];  
    const subHeader \= this.serializeSubmessageHeader(XRCE\_SUBMSG\_TYPES.CREATE\_CLIENT, 0x01, payload.length);  
    const frame \= \[...header.bytes, ...subHeader, ...payload\];  
    return {  
      type: 'CREATE\_CLIENT',  
      streamId: 0x00,  
      sequence: header.sequence,  
      wireBytes: frame,  
      hexDump: frame.map(b \=\> b.toString(16).padStart(2, '0').toUpperCase()).join(' ')  
    };  
  }

  /\*\*  
   \* Constructs CREATE submessage (creates Participant, Topic, DataWriter, etc.)  
   \*/  
  serializeCreate(requestId, objectId, objectKind, representationXmlOrString) {  
    const header \= this.serializeHeader(0x80); // Reliable stream 128  
    const strBytes \= Array.from(new TextEncoder().encode(representationXmlOrString));  
      
    // Payload: RequestId (2B) \+ ObjectId (2B) \+ ObjectKind (1B) \+ StringLength (2B) \+ String  
    const payload \= \[  
      requestId & 0xff,  
      (requestId \>\> 8\) & 0xff,  
      objectId & 0xff,  
      (objectId \>\> 8\) & 0xff,  
      objectKind,  
      strBytes.length & 0xff,  
      (strBytes.length \>\> 8\) & 0xff,  
      ...strBytes  
    \];  
      
    const subHeader \= this.serializeSubmessageHeader(XRCE\_SUBMSG\_TYPES.CREATE, 0x01, payload.length);  
    const frame \= \[...header.bytes, ...subHeader, ...payload\];  
    return {  
      type: 'CREATE',  
      requestId: requestId,  
      objectId: objectId,  
      objectKind: objectKind,  
      streamId: 0x80,  
      sequence: header.sequence,  
      wireBytes: frame,  
      hexDump: frame.map(b \=\> b.toString(16).padStart(2, '0').toUpperCase()).join(' ')  
    };  
  }

  /\*\*  
   \* Constructs WRITE\_DATA submessage with CDR serialized payload  
   \*/  
  serializeWriteData(requestId, objectId, cdrPayloadBytes, streamId \= 0x80) {  
    const header \= this.serializeHeader(streamId);  
      
    // Payload: RequestId (2B) \+ ObjectId (2B) \+ CDR Data  
    const payload \= \[  
      requestId & 0xff,  
      (requestId \>\> 8\) & 0xff,  
      objectId & 0xff,  
      (objectId \>\> 8\) & 0xff,  
      ...cdrPayloadBytes  
    \];

    const subHeader \= this.serializeSubmessageHeader(XRCE\_SUBMSG\_TYPES.WRITE\_DATA, 0x01, payload.length);  
    const frame \= \[...header.bytes, ...subHeader, ...payload\];  
    return {  
      type: 'WRITE\_DATA',  
      requestId: requestId,  
      objectId: objectId,  
      streamId: streamId,  
      sequence: header.sequence,  
      payloadSize: cdrPayloadBytes.length,  
      wireBytes: frame,  
      hexDump: frame.map(b \=\> b.toString(16).padStart(2, '0').toUpperCase()).join(' ')  
    };  
  }

  /\*\*  
   \* Simple CDR Serializer for typical ROS 2 message fields (int32, float64, string)  
   \*/  
  static serializeCDR(fields \= {}) {  
    const buffer \= \[\];  
      
    // CDR encapsulation header: 4 bytes (0x00, 0x01, 0x00, 0x00 \= Little-Endian PLAIN\_CDR)  
    buffer.push(0x00, 0x01, 0x00, 0x00);

    for (const \[key, val\] of Object.entries(fields)) {  
      if (typeof val \=== 'number') {  
        if (Number.isInteger(val)) {  
          // 4-byte int32  
          while (buffer.length % 4 \!== 0\) buffer.push(0x00);  
          buffer.push(val & 0xff, (val \>\> 8\) & 0xff, (val \>\> 16\) & 0xff, (val \>\> 24\) & 0xff);  
        } else {  
          // 8-byte float64  
          while (buffer.length % 8 \!== 0\) buffer.push(0x00);  
          const fBuf \= new ArrayBuffer(8);  
          new Float64Array(fBuf)\[0\] \= val;  
          const u8 \= new Uint8Array(fBuf);  
          for (let b of u8) buffer.push(b);  
        }  
      } else if (typeof val \=== 'string') {  
        while (buffer.length % 4 \!== 0\) buffer.push(0x00);  
        const strBytes \= Array.from(new TextEncoder().encode(val));  
        const strLen \= strBytes.length \+ 1; // null-terminated  
        buffer.push(strLen & 0xff, (strLen \>\> 8\) & 0xff, (strLen \>\> 16\) & 0xff, (strLen \>\> 24\) & 0xff);  
        buffer.push(...strBytes, 0x00);  
      }  
    }  
    return buffer;  
  }  
}

if (typeof module \!== 'undefined' && module.exports) {  
  module.exports \= {  
    MicroXRCEEngine,  
    XRCE\_SUBMSG\_TYPES,  
    XRCE\_OBJECT\_KINDS,  
    XRCE\_STREAM\_IDS  
  };  
}  
