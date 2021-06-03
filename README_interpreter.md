### INTRODUCTION:

The Monero Node GUI uses Automate to control monerod through Termux and the Termux:Tasker plugin based on device conditions and a simple notification based UI.

One downside of Automate is that the free version works only with 30 blocks or fewer and the paid version is only available through Google Playstore. To implement all the features I wanted and stay within this limit, I build a simple interpreter, which reads instructions from an external .json config file and can re-use most blocks. This config file also provides great flexibility with minimum effort.

This documentation originated from my notes and briefly outlines how the interpreter works.


### KEYWORDS:

**dict:**       loaded at startup from .json file and contains all variables and scripts
**value:**      a value can be a literal or a variable
**literal:**    a fixed value stored directly in the commands of scripts
**variable:**   a value in the root level of the dict, accessed by their key
**key:**        unique identifier that must only contain letters \[a-z,A-Z\], numbers \[0-9\] and underscores \[_\]

**flow:**       in automate a script is called "flow"
**fiber:**      in automate threads are called "fibers"
**script:**     the flow can execute conditional user defined actions, called "scripts"
**async:**      the flow uses multiple asynchronous fiberes to show the notifiaction and react to device conditions
**noti:**       prefix for data used by the notification UI
**script:**     independent blocks of code consistion of one or more commands

**command:**    each command can do simple tasks, with one or more of the following steps executed in order
**fetch:**      loads the next line unless break condition is true
**decode:**     loads variables and/or literals into a tmp register and optionally perform operations on them
**execute:**    uses the prepared tmp register for an action and writes the optional return value to the out register
**store:**      stores literals, variables, registers or async return values into the dict
**toast:**      show a non-blocking toast overlay popup, useful for debugging


### Access Data

To keep the code in the flow fairly short, some keys the the config file had to be abreviated. To keep it simple the keys are always the first letter of the phase + first letter of the usage. ALL values are optional.
```
fo,fx,fy,fz:  fetch   - <value fx> <operand fo> <value fy> ? continue script : break
do,dx,dy,dz:  decode  - reg_tmp = <value dx> <operand do> <value dy> (some functions may use value dz)
eo,ex:        execute - do <instruction eo> until reg_out contains <ex>, use reg_tmp and specific variables
sk,sx,sr:     store   - store <value sx> OR <register sr> to <key sk> (key optional for storing register values)
   tx:        toast   - toast message with value <tx ++ ty>
```

By default, all values are literals:
```
    number eg. 123, 420.69, 0xC0FE, 0b01000101
    string eg. "hello world"; escape sequences supported: \b,\f,\n,\r,\t,\',\",\\,\{,\uXXXX
    array  eg. \[123, "hello world", \["nested", "a", "b"\]]
    dict   eg. {"key1":"val1", "key2":"val2", "key3":{"keyA":"intA", "keyB":"intB"}}
```

Alternatively you can load ANY variable by appending a "$" to the key in the config file. For example "fx":"async_idx" will use the literal "async_idx"; "fx$":"async_idx" will load the value from the variable with the key "async_idx". This ONLY works for the two-letter keys listed above.


### Command Overview

For all logic, Automate consideres the following values as "false" (and everything es "true")
```
    null    (no value)
    0       (number 0)
    ""      (empty string)
    []      (empty array)
    {}      (empty dictionary)
```

Currently implemented operands are:
```
    null:     load value x (null = operand not specified)
    "[]":     load named value 
    "[][]":   load named nested value
    "pack":   create a new dictionary from variables, pass an array of keys to select which
    "type":   get type as sting
    "type=":  compare the type of to variables, 1 if identical
    "type!=": compare the type of to variables, 0 if identical same
    "+":      addition
    "-":      substraction
    "*":      multiplication
    "/":      division
    "//":     integer division. rounds down
    "%":      modulo
    "+1%":    increment, then modulo
    "+1%#":   increment, then modulo of length. Useful to wrap-around the idex of an array at it's end.
    "~":      binary invert
    "&":      binary and
    "|":      binary or
    "^":      binary xor
    "<<":     binary shift left
    ">>":     binary shift right
    ">>>":    binary shift right
    "=":      equals, can be used to compare strings
    "!=":     equals not, can be used to compare strings
    ">":      greater than
    ">=":     greater or euqal
    "<":      less than
    "<=":     less or equal
    "!":      logic invert
    "&&":     logic and   , returns b if true, a if false
    "||":     logic or    , returns a if true, b if false
    "++":     concat strings
    "str":    convert to string
    "num":    convert to number
    "max":    returns the larger value of two inputs
    "jsonEn": encode a dictionary into a json formatted string
    "jsonDe": decode a json formatted string into a dictionary
    "time":   format a unix timestamp according to a formatting pattern. Leave the timestamp empty to use "Now"
    "dura":   format a duration between two timestamps according to a formatting pattern. Leave either timestamp empty to use "Now"
    "cont":   retruns 1 if a string constains a substring or a dictionary contains a key. dz parameter are optional flags
    "!!find": returns 1 if a within a string a regex pattern matched, else returns 0
    "replace":replaces parts of a string based on a regex search pattern with a regex replace pattern
    "strByte":custom formatting function for any data size. If there are mor than 3 digits infront of the comma it switches to the next unit: B,KB,MB,GB,TB
    "strMet": custom formatting function for any metric data. Automatically scales the value and appends a K,M,G,T if necassary
```

```
Currently implemented instructions:
    null  : skip (null = instruction not specified)
    "noti": taps notification to reload values
    "give": passes value to notification, "reg_tmp" must be a dictionary with all values
    "term": runs command "monerod <reg_tmp>" in termux; returns "reg_out"
    "text": opens user input dialog; returns "reg_out"
    "menu": opens user menu dialog; returns "reg_out"
    "http": http request, can be used to access local RPC node
```

### VALUES IN BLOCKS:

FETCH:
```
!((script_key[0] != "/"[0]) &&
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = null        ? (cv["fx"]!=null? cv["fx"] : cv["fx$"] != null ? dict[cv["fx$"]] : 1)                            :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "[]"        ? (cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])[(cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])]        :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "[][]"    ? (cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])[(cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])][(cv["fz"]!=null? cv["fz"] : dict[cv["fz$"]])]    :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "type="    ? type(cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])=    (cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])     :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "type!="    ? type(cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])!=(cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])     :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "~"        ? ~(cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])                                                        :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "&"        ? (cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])    &    (cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])    :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "|"        ? (cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])    |    (cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])    :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "^"        ? (cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])    ^    (cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])    :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "="        ? (cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])    =    (cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])    :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "!="        ? (cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])    !=    (cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])    :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = ">"        ? (cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])    >    (cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])    :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = ">="        ? (cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])    >=    (cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])    :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "<"        ? (cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])    <    (cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])    :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "<="        ? (cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])    <=    (cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])    :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "!"        ? !(cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])                                                        :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "&&"        ? (cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])    &&    (cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])    :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "||"        ? (cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]])    ||    (cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]])    :
(cv["fo"]!=null? cv["fo"] : dict[cv["fo$"]]) = "cont"    ? contains(cv["fx"]!=null? cv["fx"] : dict[cv["fx$"]], cv["fy"]!=null? cv["fy"] : dict[cv["fy$"]], cv["fz"]!=null? cv["fz"] : dict[cv["fz$"]]):
null)
```


DECODE:
```
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = null        ? (cv["dx"]!=null? cv["dx"] : cv["dx$"] != null ? dict[cv["dx$"]] : 1)                                :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "[]"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])[(cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])]        :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "[][]"    ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])[(cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])][(cv["dz"]!=null? cv["dz"] : dict[cv["dz$"]])]    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "?"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])? (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]]) : (cv["dz"]!=null? cv["dz"] : dict[cv["dz$"]]):
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "pack"    ? sift(dict,(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]]))                                             :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "type"    ? type(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])                                                    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "type="    ? type(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])=    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])     :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "type!="    ? type(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])!=(cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "+"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    +    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "-"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    -    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "*"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    *    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "/"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    /    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "//"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    //    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "%"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    %    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "+1%"    ?((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])+1)%    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "+1%#"    ?((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])+1)%(#(cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]]))    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "~"        ?~(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])                                    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "&"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    &    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "|"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    |    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "^"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    ^    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "<<"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    <<    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = ">>"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    >>    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = ">>>"    ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    >>>    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "="        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    =    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "!="        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    !=    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = ">"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    >    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = ">="        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    >=    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "<"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    <    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "<="        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    <=    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "!"        ? !(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])                                :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "&&"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    &&    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "||"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    ||    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "++"        ? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])    ++    (cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]])    :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "str"    ?++(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])                                                        :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "num"    ?+(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])                                                        :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "max"    ? ((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]]||0)>(cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]]||0))? (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]]):(cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]]):
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "jsonEn"    ? jsonEncode(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])                                            :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "jsonDe"    ? jsonDecode(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])                                            :
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "time"    ? dateFormat(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]]||Now, cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]]):
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "dura"    ? durationFormat((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]]||Now)-(cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]]||Now),(cv["dz"]!=null? cv["dz"] : dict[cv["dz$"]]||null)):
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "cont"    ? contains(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]], cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]], cv["dz"]!=null? cv["dz"] : dict[cv["dz$"]]):
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "!!find"    ? !!findAll(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]], cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]]):
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "replace"? replaceAll(cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]], cv["dy"]!=null? cv["dy"] : dict[cv["dy$"]], cv["dz"]!=null? cv["dz"] : dict[cv["dz$"]]):
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "strByte"? replaceAll((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]]) >= 1073741824000? ((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])/1099511627776)++"TB" : (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]]) >= 1048576000? ((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])/1073741824)++"GB" : (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]]) >= 1024000? ((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])/1048576)++"MB" : (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]]) >= 1000? ((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])/1024)++"KB" : (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])++"B", "^(\\d(?:\\.?\\d)\{0,3\})\\d*(\\D*)$", "$1$2"):
(cv["do"]!=null? cv["do"] : dict[cv["do$"]]) = "strMet" ? replaceAll((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]]) >= 1000000000000? ((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])/1000000000000)++"T" : (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]]) >= 1000000000? ((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])/1000000000)++"G" : (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]]) >= 1000000? ((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])/1000000)++"M" : (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]]) >= 1000? ((cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])/1000)++"K" : (cv["dx"]!=null? cv["dx"] : dict[cv["dx$"]])++"", "^(\\d(?:\\.?\\d)\{0,3\})\\d*(\\D*)$", "$1$2"):
null
```



EXECUTE:
```
!(cv["eo"] || dict[cv["eo$"]])        || type(reg_out) != "number" &&
findAll(reg_out, cv["ex"] != null? cv["ex"]: cv["ex$"]!= null? dict[cv["ex$"]] : "^")

(cv["eo"] || dict[cv["eo$"]]) = "give"
(cv["eo"] || dict[cv["eo$"]]) = "noti"
(cv["eo"] || dict[cv["eo$"]]) = "term"
(cv["eo"] || dict[cv["eo$"]]) = "http"
(cv["eo"] || dict[cv["eo$"]]) = "menu"
```

NOTI ACTION:
```
dict["async_noti"]? cv["ex"] || dict[cv["ex$"]] || 0x2 : 0x4
```

NOTI ID:
```
dict["async_noti"] || null
```


EXECUTE_DATA_FIELDS:
```
cv["fail_rtry"]!=null? cv["fail_rtry"] : cv["fail_rtry$"] != null ? dict[cv["fail_rtry$"]] : dict["fail_rtry"]

cv["menu_mult"]!=null? cv["menu_mult"] : cv["menu_mult$"] != null ? dict[cv["menu_mult$"]] : dict["menu_mult"]
cv["http_mthd"]!=null? cv["http_mthd"] : cv["http_mthd$"] != null ? dict[cv["http_mthd$"]] : dict["http_mthd"]
cv["http_type"]!=null? cv["http_type"] : cv["http_type$"] != null ? dict[cv["http_type$"]] : dict["http_type"]
jsonEncode(cv["http_body"]!=null? cv["http_body"] : cv["http_body$"] != null ? dict[cv["http_body$"]] : dict["http_body"])
cv["http_time"]!=null? cv["http_time"] : cv["http_time$"] != null ? dict[cv["http_time$"]] : dict["http_time"]

cv["menu_head"]!=null? cv["menu_head"] : cv["menu_head$"] != null ? dict[cv["menu_head$"]] : dict["menu_head"]
cv["menu_list"]!=null? cv["menu_list"] : cv["menu_list$"] != null ? dict[cv["menu_list$"]] : dict["menu_list"]
cv["menu_mult"]!=null? cv["menu_mult"] : cv["menu_mult$"] != null ? dict[cv["menu_mult$"]] : dict["menu_mult"]
cv["menu_canc"]!=null? cv["menu_canc"] : cv["menu_canc$"] != null ? dict[cv["menu_canc$"]] : dict["menu_canc"]
cv["menu_sort"]!=null? cv["menu_sort"] : cv["menu_sort$"] != null ? dict[cv["menu_sort$"]] : dict["menu_sort"]
cv["menu_pres"]!=null? cv["menu_pres"] : cv["menu_pres$"] != null ? dict[cv["menu_pres$"]] : dict["menu_pres"]

cv["text_head"]!=null? cv["text_head"] : cv["text_head$"] != null ? dict[cv["text_head$"]] : dict["text_head"]
cv["text_type"]!=null? cv["text_type"] : cv["text_type$"] != null ? dict[cv["text_type$"]] : dict["text_type"]
cv["text_hint"]!=null? cv["text_hint"] : cv["text_hint$"] != null ? dict[cv["text_hint$"]] : dict["text_hint"]
cv["text_body"]!=null? cv["text_body"] : cv["text_body$"] != null ? dict[cv["text_body$"]] : dict["text_body"]
```

STORE KEY:
```
(cv["sk"]!=null? cv["sk"] : dict[cv["sk$"]]) || (cv["sr"]!=null? cv["sr"] : dict[cv["sr$"]]) ||null
```

STORE VAL:
```
cv["sx"]    != null ? cv["sx"]            :
cv["sx$"]    != null ? dict[cv["sx$"]]    :
(cv["sr"]!=null? cv["sr"] : dict[cv["sr$"]])    = "reg_tmp"        ? reg_tmp        :
(cv["sr"]!=null? cv["sr"] : dict[cv["sr$"]])    = "reg_out"        ? reg_out        :
(cv["sr"]!=null? cv["sr"] : dict[cv["sr$"]])    = "async_noti"    ? async_noti    :
(cv["sr"]!=null? cv["sr"] : dict[cv["sr$"]])    = "async_num"    ? async_num        :
(cv["sr"]!=null? cv["sr"] : dict[cv["sr$"]])    = "async_dict"    ? async_dict    :
(cv["sr"]!=null? cv["sr"] : dict[cv["sr$"]])    = "async_idx"    ? async_idx        :
(cv["sr"]!=null? cv["sr"] : dict[cv["sr$"]])    = "async_key"    ? async_key        :
(cv["sr"]!=null? cv["sr"] : dict[cv["sr$"]])    = "async_arr"    ? async_arr        :
null
```

TOAST:
```
cv["tx"] != null ? cv["tx"] : cv["tx$"] != null ? dict[cv["tx$"]] : null
```