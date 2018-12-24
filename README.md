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
        schema type: tlv, w_tag: 8, w_len: 16 do
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
        schema type: tlv, w_tag: 8, w_len: 16 do
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
