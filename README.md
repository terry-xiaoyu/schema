# schema
DSL for converting between binaries and maps

## Getting started
**Schema** is a DSL created for parsing a binary to elixir map or in opposite direction, reverting the map back to binary. It does the parsing/unparsing according to the `schemas` defined.
Schema provides serveral primitives:
- parse
- unparse
- schema
- tlv

## Exmaples
#### Simple parse
```
import Schema

def decode(payload) do
    parse payload do
        ## the content of the payload is of tlv format,
        ## the first 1 bytes contains the `tag` of the tlv,
        ## the next 2 bytes contains the length of the value,
        ## and following the length field is the number of bytes specified by the length.
        schema type: tlv, w_tag: 1, w_len: 2 do
            tlv tag: 0x01, type: "int", as: "timestamp"
            tlv tag: 0x02, type: "string", as: "event"
        end
    end
end
```

Given
```
payload = <<1, 4, 92,32,170,241, 2, 5, 104,101,108,108,111>>
```
Then the function call `decode(payload)` would produce the following map:
```
%{
    "timestamp" => 1545644785,
    "event" => "hello"
 }
```

If the timestamp is not present in the payload:
```
payload = <<2, 5, 104,101,108,108,111>>

```
Then the result shall be:
```
%{
    "event" => "hello"
 }
```
#### Simple unparse
Unparse is the reverse of the parse:

```
import Schema

def encode(payload) do
    unparse payload do
        schema type: tlv, w_tag: 1, w_len: 2 do
            tlv tag: 0x01, type: "int", as: "timestamp"
            tlv tag: 0x02, type: "string", as: "event"
        end
    end
end
```
Given
```
payload =
  %{
    "timestamp" => 1545644785,
    "event" => "hello"
  }
```
Then the function call `decode(payload)` would produce the following binary:
```
<<1, 4, 92,32,170,241, 2, 5, 104,101,108,108,111>>
```
