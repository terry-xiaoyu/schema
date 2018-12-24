# schema
DSL for converting between binaries and maps

## Getting started
**Schema** is a DSL created for parsing a binary to elixir map or in opposite direction, reverting the map back to binary. It does the parsing/unparsing according to the `schemas` defined.
Schema provides serveral primitives:
- parse: parse a binary to map.
- unparse: unparse a map to binary.
- schema: the schema for a data field. A field defined by `schema` MUST be present in the payload. The definiation order of `schema` primitives matches the order of the fields in the binary. `schema` is for both parse and unparse.
- tlv: the tlv schema for a data field. A field defined by `tlv` MAY be present in the payload, i.e. it is optional. And the order is undefined in the binary. `schema` is for both parse and unparse.

## Exmaples
#### Simple parse
```elixir
import Schema

def decode(payload) do
    parse payload do
        ## the content of the payload is of tlv format,
        ## the first 1 bytes contains the `tag` of the tlv,
        ## the next 2 bytes contains the length of the value,
        ## and following the length field is the number of bytes specified by the length.
        schema type: tlv, w_tag: 1, w_len: 2 do
            ## two kinds of tlvs are defined:
            ##   - tag = 0x01, the value will be parsed as an integer with a key = "timestamp"
            ##   - tag = 0x02, the value will be parsed as a string with a key = "event"
            tlv tag: 0x01, type: "int", as: "timestamp"
            tlv tag: 0x02, type: "string", as: "event"
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
#### Simple unparse
Unparse is the reverse of the parse:

```elixir
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
