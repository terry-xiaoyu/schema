# schema
DSL for converting between binaries and maps

## Getting started
**Schema** is a DSL created for parsing a binary to elixir map or in opposite direction, reverting the map back to binary. It does the parsing/unparsing according to the `schemas` defined.
Schema provides serveral primitives:
- parse: parse a binary to map.
- unparse: unparse a map to binary.
- schema: the schema for a data field. The definiation order of `schema` primitives matches the order of the fields in the binary. `schema` is for both parse and unparse.

## Exmaples
### Simple parse
```elixir
import Schema

def decode(payload) do
    parse payload do
        ## the content of the payload is of tlv format,
        ## the first 1 bytes contains the `tag` of the tlv,
        ## the next 2 bytes contains the length of the value,
        ## and following the length field is the number of bytes specified by the length.
        schema type: "tlv-spec", width_tag: 1, width_len: 2 do
            ## two kinds of tlvs are defined:
            ##   - tag = 0x01, the value will be parsed as an integer with a key = "timestamp"
            ##   - tag = 0x02, the value will be parsed as a string with a key = "event"
            schema type: "tlv", tag: 0x01, type: "int", as: "timestamp"
            schema type: "tlv", tag: 0x02, type: "string", as: "event"
        end
    end
end
```

Given
```elixir
payload = <<1, 4, 92,32,170,241, 2, 5, 104,101,108,108,111>>
```
Then the function call `decode(payload)` would produce the following map:
```elixir
%{
    "timestamp" => 1545644785,
    "event" => "hello"
 }
```

If the timestamp is not present in the payload:
```elixir
payload = <<2, 5, 104,101,108,108,111>>

```
Then the result shall be:
```elixir
%{
    "event" => "hello"
 }
```
### Simple unparse
Unparse is the reverse of the parse:

```elixir
import Schema

def encode(payload) do
    unparse payload do
        schema type: "tlv-spec", width_tag: 1, width_len: 2 do
            schema type: "tlv", tag: 0x01, type: "int", as: "timestamp"
            schema type: "tlv", tag: 0x02, type: "string", as: "event"
        end
    end
end
```
Given
```elixir
payload =
  %{
    "timestamp" => 1545644785,
    "event" => "hello"
  }
```
Then the function call `decode(payload)` would produce the following binary:
```elixir
<<1, 4, 92,32,170,241, 2, 5, 104,101,108,108,111>>
```


### Now a non-trivial example

Suppose we have a protocol defined as following:

|Field | Version | SequenceNO | Reserved | MsgType | SubMsgType | Data    | CRC32 |
|------|---------|------------|----------|---------|------------|---------|-----|
|Length| 1 Byte  | 2 Bytes    | 2 bits   | 2 bits  | 12 bits    | n Bytes | 4 Bytes|

Where the `Data` field consists of 0, one or many TLVs with common structure:
```
Tag: 2 Bytes
Len: 2 Bytes
Value: Len Bytes
```

And the `Value` field of the TLVs depends on the `Tag` field:

- When `Tag` == 0x01, it is a 'alarm' message:

|Field | Length  | Type |
|------|---------|------|
|UTC   | 1 Byte  | Integer|
|IsHighAlarm | 1 Byte  | Boolean|
|Msg   | -      | String|

- When `Tag` == 0x02, it is "data reports", which type is defined by the bits in the first Byte:
  Every bit in the first byte indicates the presence of the followed fields:

|Field | Length  | Type |
|------|---------|------|
|Flag   | 1 Byte  | Integer|
|Temperature | 1 Byte | Integer |
|Humidity    | 1 Byte | Integer |

- When `Tag` == 0x03, it is a 'device info' message, contains following 3 white-space separated fields:

|Field | Length  | Type |
|------|---------|------|
|Manufacturer   | -  | String|
|ProductID      | -  | String|
|DeviceID       | -  | String|

```elixir
import Schema

def decode(payload) do
    parse payload, endian: big do
        schema len: 1, type: "int", as: "version"
        schema len: 2, type: "int", as: "sequence_no"
        schema len: 2-bits, type: "int", as: "reserved", hidden: true
        schema len: 2-bits, type: "int", as: "message_type"
        schema len: 12-bits, type: "int", as: "sub_message_type"
        schema type: "tlv-spec", width_tag: 2, width_len: 2, as: "data" do
            schema type: "tlv", tag: 0x01, as: "alarm" do
                schema len: 1, type: "int", as: "utc_time"
                schema len: 1, type: "boolean", as: "is_high_alarm"
                schema type: "string", as: "msg"
            end
            schema type: "tlv", tag: 0x02, as: "data_reports" do
                schema len: 1, type: "int", as: "flag", hidden: true
                schema len: 1, type: "int", as: "temperature" do
                    depends_on_bit(1, flag)
                end
                schema len: 1, type: "int", as: "humidity" do
                    depends_on_bit(2, flag)
                end
            end
            schema type: "tlv", tag: 0x03, as: "device_info" do
                [Manufacturer, ProductID, DeviceID] = String.split(device_info)
                %{
                    "manufacturer" => Manufacturer,
                    "product_id" => ProductID,
                    "device_id" => DeviceID
                }
            end
        end
        schema len: 4, type: int, as: "crc32"
    end
end
```

**TO DO**
Given
```elixir
payload = xxx
```
**TO DO END**

Then the function call `decode(payload)` would produce the following map:

```elixir
%{
    "version" => 1,
    "sequence_no" => 106,
    "message_type" => 2,
    "sub_message_type" => 32,
    "data" => %{
        "alarm" => %{
            "utc_time" => 1545644785,
            "is_high_alarm" => true,
            "msg" => "low battery"
        },
        "data_reports" => % {
            "temperature" => 35
        },
        "device_info" => %{
            "manufacturer" => "EMQX",
            "product_id" => "Product1",
            "device_id" => "Device1"
        }
    }
}
```
