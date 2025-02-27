# Supported Filters

Quex Request Oracles support the subset of [jq](https://jqlang.github.io/jq/manual/) language for response post-processing. The supported operations are

+ `+`, `-`, `*`, `/`, `%`
+ Selecting field by key or index via `.` or `[]`
+ Array slicing `[n:m]`
+ Pipe `|`
+ Array construction `[]`
+ `map`
+ `floor`, `abs`, `round`, `sqrt`
+ `split`, `join`
+ `todate`, `fromdate`
+ `tonumber`
