# Supported Jq Expressions

Quex Oracle supports the subset of [Jq](https://jqlang.github.io/jq/manual/) language for response post-processing. The supported operations are

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
