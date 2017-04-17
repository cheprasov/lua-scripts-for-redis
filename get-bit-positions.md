# GET-BIT-POSITIONS
Current script finds setted bits by a key with a specified range.
I think, this script is more useful than [BITPOS](https://redis.io/commands/bitpos) in [Redis](https://redis.io)

## SCRIPT key offset limit
- key - a key in redis storage. KEYS[1]
- offset - the offset is interpreted as count of bits (not bytes). ARGV[1]
- count - max count of list with positions (offsets) of bit 1. ARGV[2]

Returns a list with the positions (offset in bits) or empty list, if nothing found.

## PHP Example

```php
<?php

include('./vendor/autoload.php');

use RedisClient\ClientFactory; // https://github.com/cheprasov/php-redis-client

$Redis = ClientFactory::create();

// Setup bits
$Redis->set('bits', chr(0b11011011) . chr(0b00100100) . chr(0b10101010));

$script = '<script here>';
$res = $Redis->evalScript($script, ['bits'], [7, 10]);
var_export($res);
```
result:
```php
array(
  0 => 7,
  1 => 10,
  2 => 13,
  3 => 16,
  4 => 18,
  5 => 20,
  6 => 22,
);
```

## LUA script

```lua
local offset = tonumber(ARGV[1]);
local limit = tonumber(ARGV[2]);
local offset_byte = math.floor(offset / 8);
local ids = {};
local cache = {};

for i = 1, limit do

    if (#ids == limit) then
        break;
    end;

    local pos = redis.call("BITPOS", KEYS[1], 1, offset_byte);

    if (pos == -1) then
        break;
    end;

    cache[pos] = 1;

    if (pos < offset) then
        pos = offset;
    end;

    offset_byte = math.floor(pos / 8) + 1;

    local est;
    local offset_bit = offset_byte * 8 - 1;

    for j = pos, offset_bit do
        if (#ids == limit) then
            break;
        end;
        if (cache[j]) then
            ids[#ids + 1] = j;
        else
            est = redis.call("GETBIT", KEYS[1], j);
            if (est == 1) then
                ids[#ids + 1] = j;
            end;
        end;
    end;

end;

return ids;
```
