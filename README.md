# VI-feeder-firmware
## Vet Innovations Control firmware

Pre-compiled firmware binary for the Vet Innovations pet feeder system.  Written in C.

### Outgoing Message Syntax
The pet feeder transmits outgoing serial messages to the Photon uC according to the following syntax:

```
  0x02 A B XXXXXXXXXXXX....XXX 0x03
```

This means messages are framed by two characters.  (Of course the actual serial data does not include spaces between characters, and the hex codes must be cast as chars).
*  0x02 = ASCII 'Start Text' char
*  0x03 = ASCII 'End Text' char

Any messages not framed between these chars will be discarded by the Photon.

The first two bytes (`AB`) of any serial message are referred to as the message header and they describe the purpose and function of the message:
*  `A` describes the category of the message.  I.E. `A = 'E'` for messages about errors, `A='F'` for messages about feeding, etc.  (See below)
*  `B` describes the specific nature of the message.  (See below)

The rest of the message (`XXX...XXX`) can be any length, and should be a stringified JSON object, suitable for interpretation by the cloud.  JSON is just a dictionary; a type of syntax for constructing maps or hashes, used frequently in web development.  An example JSON object in JavaScript might look like this:

```
{
  key1: "value1",
  key2: "value2",
  key3: {
    key3_1: "value3"
  }
}
```

These can be nested arbitrarily, and the actual literal value of the keys can be anything.  However, when a JSON object is "stringified," it means that the object is just escaped.  So, the above object would become:

```
"key1: \"value1\", key2: \"value2\", key3: {key3_1: \"value3\"}"
```

The Photon would expect the payload of outgoing messages (`XXX...XXX`) to take this form.  **Notice that we are also stripping the beginning { and ending }.**  See below for specific examples for each type of event.

### Outgoing Messages - Reference
#### Feedings
The cloud needs to know when a feeder dispenses food; the datetime and quantity of this food are the two most important things here.

#### Pet Presence
##### Pet Arrives
When an RFID tag appears within range, we need to report this new pet arrival to the cloud.  This message must contain a UTC timestamp (as an int), as well as the data about the pet that arrived (name and RFID).

`AB` header: `PA`

JSON Payload:

```
{
  ts: 123456789,
  pet: {
    name: "Alice",
    rfid: "AAAAAAAAAA"
  }
}
```

Serial message expected from Microchip uC:

`char[] exampleMsg = char(0x02) + "PAts:123456789,pet:{name:\"Alice\",rfid:\"AAAAAAAAAA\"}" + char(0x03);`


##### Pet Departs
When an RFID tag disappears from the feeder's detection range for a predefined period of time (~3 ping cycles), we need to report this pet departure to the cloud.  This message must contain a UTC timestamp (as an int), as well as the data about the pet that left (name and RFID).

`AB` header: `PD`

JSON Payload:

```
{
  ts: 123456789,
  pet: {
    name: "Alice",
    rfid: "AAAAAAAAAA"
  }
}
```

Serial message expected from Microchip uC:

`char[] exampleMsg = char(0x02) + "PDts:123456789,pet:{name:\"Alice\",rfid:\"AAAAAAAAAA\"}" + char(0x03);`


---

#### RFID List
The cloud will need to know what pets have been registered to a feeder, as well as the corresponding RFID of those pets.  It's recommended that the serial transmission of this message be wrapped in a method so it can be easily called from anywhere in the feeder code, such as on boot but also on request from the cloud.

`AB` header: `RL`

JSON Payoad:

```
{
  pets: [
    {
      name: "Alice",
      rfid: "AAAAAAAAAA"
    },
    {
      name: "Bob",
      rfid: "BBBBBBBBBB"
    }
  ]
}
```

Serial message expected from Microchip uC:

`char[] exampleMsg = char(0x02) + "RLpets:[{name:\"Alice\",rfid:\"AAAAAAAAAA\"},{name:\"Bob\",rfid:\"BBBBBBBBBB\"}]" + char(0x03);`

---

#### Errors
