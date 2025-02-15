# gmod-lua-performance
A simple comparison of performance optimizations for gLUA

These are a few benchmarks that are tested on a Garry's Mod Server idling with DarkRP with only 1 player online.
Some of those benchmarks, if possible, were also tested in a LUA5.3 console on Debian 10. If not otherwise stated differently all tests have been done on a 32bit gmod server on linux.  

These benchmark results mean nothing compared to other benchmark results. Your numbers will probably look completely different than my numbers. Please compare your benchmark results only with your own numbers, as every small change in Hardware changes the result notably.

**The goal of this collection of benchmarks is to show the actual performance gain by implementing "Coding Tips" from the web.**

**Everyone can test the code for themselves and see the difference. Source code is always linked at the bottom of a comparison.**

The sources for some of the "Coding Tips" are:

[lua-users.org](http://lua-users.org/wiki/OptimisationCodingTips)  
[lua.org](https://www.lua.org/gems/sample.pdf)  
[stackoverflow.com](https://stackoverflow.com/questions/154672/what-can-i-do-to-increase-the-performance-of-a-lua-program/12865406#12865406)  

If you want to read more about gmod lua then you could also visit my wiki at [luctus.at/wiki](https://luctus.at/wiki/)


# Foreword

Garry's Mod doesn't use only lua. It uses lua-jit.  
This means the following benchmarks show, where applicable, also a comparison of lua and lua-jit.  
Many of the performance tips online are supposed to be for lua, not luajit.

An example of lua vs lua-jit with the "multiplication vs division" example:

    $ time luajit-2.1.0-beta3 mult_vs_div.lua
    Multiplication: 0.00111624
    Division: 0.00110691

    real    0m1.255s
    user    0m1.054s
    sys     0m0.182s

    $ time lua5.3 mult_vs_div.lua
    Multiplication: 0.0284177
    Division: 0.03088507

    real    0m21.773s
    user    0m21.623s
    sys     0m0.120s

If you find any error, mistake or completely different result on your side please create a pull request.  
I want this page to be as accurate as possible so that everyone can see and learn what actually benefits your code's performance.



# caching functions in HUDPaint

TL;DR: If you use a a function, e.g. LocalPlayer(), often in a HUDPaint hook then I highly suggest caching it once at the beginning.

This is a comparison of the following code bits. The question is: "Should you cache a function you only use 3 times?".

The unoptimized version:

```lua
draw.RoundedBox(0, LocalPlayer():Health(), 500, 500, 500, color_white )
draw.RoundedBox(0, LocalPlayer():Health(), 200, 200, 200, color_white )
draw.RoundedBox(0, LocalPlayer():Health(), 10, 10, 10, color_white )
```

and the better version:

```lua
local lph = LocalPlayer():Health()
draw.RoundedBox(0, lph, 500, 500, 500, color_white )
draw.RoundedBox(0, lph, 200, 200, 200, color_white )
draw.RoundedBox(0, lph, 10, 10, 10, color_white )
```

The result (10000 frametimes):

    Non-Local:  0.00001756282000116   (1.756282000116e-05)
    Local:      0.000006777780000516  (6.777780000516e-06)

Code: [files/hudpaint_3call_cache.lua](files/hudpaint_3call_cache.lua)



# GetNWString vs getDarkRPVar

TL;DR: getDarkRPVar is always faster on server while on client it seems to be random. I would recommend using getDarkRPVar, as it is more optimized for network usage in comparison to NW1's constant network traffic every 10 seconds.

This is a comparison between `ply:GetNWString("job")` and `ply:getDarkRPVar("job")`.

Result:

    --- Benchmark complete
    reps	10	rounds	10000
    On Server
    darkrpvar	1.2346400001078e-07
    gmodnwvar	2.0404500001689e-07
    --- Benchmark complete
    reps	10	rounds	10000
    On Client
    darkrpvar	2.4343500003823e-07
    gmodnwvar	2.036369999766e-07

As you can see, the result is different on client and server. The result on the client is always different everytime i run these tests. Sometimes, on the client , the gmodnwvar is faster by 25% and other times it is 300% slower than darkrpvar.

Code: [files/nwvar_vs_darkrpvar.lua](files/nwvar_vs_darkrpvar.lua)



# Config Table vs Variable

TL;DR: Using a table is slower than using a variable.

This is a comparison between using a config table like `myaddon.config.color =` and a simple variable like `myaddon_config_color =`. This mainly exists because some people don't believe me.

The result:

    --SERVER
    tab=	1.9999993128295e-07
    var=	1.0000007932831e-07
    --CLIENT
    tab=	1.0000007932831e-07
    var=	0

As you can see above, using a single variable instead of a table structure is faster.


## Explanation

Getting a variable is as simple as "getting" it. But with a table you have to get the table and then the table element inside.  
You can inspect this with lua's luac command, e.g. `luac5.4 -l file.lua`.  
An example: `print(abc)` vs `print(a.b.c)`

```
$ luac5.4 -l var.lua
[...]
        GETTABUP        0 0 2   ; _ENV "print"
        GETTABUP        1 0 0   ; _ENV "abc"
        CALL            0 2 1   ; 1 in 0 out
        RETURN          0 1 1   ; 0 out

$ luac5.4 -l tab.lua
[...]
        GETTABUP        0 0 4   ; _ENV "print"
        GETTABUP        1 0 0   ; _ENV "a"
        GETFIELD        1 1 1   ; "b"
        GETFIELD        1 1 2   ; "c"
        CALL            0 2 1   ; 1 in 0 out
        RETURN          0 1 1   ; 0 out
```

As you can observe above, the table variant simply has more executions and is thus slower in theory and practice as shown above.

Code: [files/table_vs_variable.lua](files/table_vs_variable.lua)



# pairs vs ipairs vs for i=1

TL;DR: They are not as different as people make it out to be. Pairs is slower because it also works on non-sequential tables.

The result (each method loops 10000 times):

    pairs:    0.0095316000000025
    ipairs:   0.0082050999999979
    for i=1:  0.0077024000000279


Code: [files/pairs_vs_ipairs_vs_for.lua](files/pairs_vs_ipairs_vs_for.lua)



# Distance vs DistToSqr

TL;DR: The time difference between Distance and DistToSqr is barely noticable.

The result (average of 10000 calculations):

    Distance:   0.00000026040000557259 (2.6040000557259e-07)
    DistToSqr:  0.00000025130000176432 (2.5130000176432e-07)

The Code: [files/distance_vs_disttosqr.lua](files/distance_vs_disttosqr.lua)



# ternary vs if

TL;DR: They take the same amount of time. The difference is randomly swinging a tiny bit to each side every test.

Tested: `a = t and 7 or 1` vs `a=1 if t then a=7 end`

The result (average of 1000000 calculations):

    if:      4.1537399886693e-08
    ternary: 4.168440006697e-08

The Code: [files/ternary_vs_if.lua](files/ternary_vs_if.lua)



# table.insert vs table[#table+1] = 

TL;DR: The time difference between using table.insert(myTable,9) and myTable[#myTable+1] = 9 is barely noticable, as the results were mostly the same. (Even if tested on Lua5.3 console)

The result (inserting 1million values):

    table.insert:         0.066328400000003
    local table.insert:   0.07047399999999
    [#res+1]:             0.066876700000023

The Code: [files/table.insert_vs_table.lua](files/table.insert_vs_table.lua)



# table.HasValue(table,x) vs table[x]

TL;DR: Directly checking the table key with table[value] is way faster.

The result (1million lookups):

    table.HasValue:   0.032319600000051
    table[value]:     0.00024150000001555


The Code: [files/table.hasvalue_vs_table.lua](files/table.hasvalue_vs_table.lua)

Note:
This is a O(n) vs O(1) situation. This means the bigger the table the slower table.HasValue will get.  
For other examples visit [https://wiki.facepunch.com/gmod/table.HasValue](https://wiki.facepunch.com/gmod/table.HasValue)  
To learn more about the "Big O Notation" visit, for example, [https://web.mit.edu/16.070/www/lecture/big_o.pdf](https://web.mit.edu/16.070/www/lecture/big_o.pdf)



# table.Empty(tab) vs tab = {}

TL;DR: Assigning an empty table is overall faster than emptying it, except if you have really small tables (~10 elements).

The following test compares small (10 elements) and big (10.000) tables with each method. It also differentiates between numerical tables (key 1,2,3,...) and string index tables (key a,b,c,...).  
Result:

    --- Benchmark complete
    reps	1	rounds	1000
    On Server
    table.Empty numerical on small	1.5931000018554e-06
    table.Empty numerical on big  	5.295060000276e-05
    table.Empty stringind on small	4.9099999841928e-07
    table.Empty stringind on big  	4.7529999847029e-07
    table = {} numerical on small	9.8249999609834e-07
    table = {} numerical on big  	6.0400000597838e-07
    table = {} stringind on small	5.8179999678032e-07
    table = {} stringind on big  	4.8149999975067e-07


The above times vary greatly by table count and amount of tests included. In general, assigning a new table is faster than emptying one. The biggest difference is between emptying a big numerical table and assigning a new one.  
An advantage of always assigning a new table: The time taken is nearly constant, so you don't have to worry about it.


The Code: [files/table_empty_vs_new.lua](files/table_empty_vs_new.lua)



# ply:GetValue(3) vs GetValue(ply,3)

TL;DR: Not using metatables is very very slightly faster.

This is a comparison of using the MetaTable PLAYER vs simply passing the player as an argument.

The result (100.000 rounds, 100 times):

    ply:GetCash(1): 0.014884074999999
    GetCash(ply,1): 0.013477668

As you can see, there is barely any speed difference between the two. Without a metamethod it is only slightly faster in every test, but not noticable enough to be a recommendation.

The code: [files/meta_vs_argument.lua](files/meta_vs_argument.lua)



# String Concat vs Table Concat

TL;DR: It is faster to add strings together via tables instead of stringing them together directly.

The result (10000 strings):

    GMOD SERVERSIDE CONSOLE
    String Concat:  0.32185959999993
    Table Concat:   0.0099451999999474
    LUA5.3 CONSOLE
    String Concat:  0.086629
    Table Concat:   0.006428

The Code: [files/string_vs_table_concat.lua](files/string_vs_table_concat.lua)



# string.format vs .. concat

TL;DR: Adding only strings together makes `..` faster, but as soon as numbers or other things come into play the `string.format` method is faster.

This is a comparison of the 2 ways of creating a single string out of multiple pieces, either with string.format or with `..` between the arguments. Example:

```lua
format = string.format("%s has steamid %s",ply:Nick(),ply:SteamID())
concat = ply:Nick().." has steamid "..ply:SteamID()
```

Result:

    --- Benchmark complete
    reps	10	rounds	1000
    On Server
    concat str	1.7449199980092e-06
    format str	2.0344400029444e-06
    concat int	2.7943200012942e-06
    format int	2.5907900007041e-06

As you can see above, the format method is slower if you only add together strings, but if you add together strings and numbers it is faster again.

The Code: [files/string_format_vs_concat.lua](files/string_format_vs_concat.lua)



# surface.DrawText vs draw.DrawText

TL;DR: Using `draw.SimpleText` is nearly as fast as using the `surface.*` functions. DrawText is a bit slower and SimpleTextOutlined is by far the slowest.

This test simply draws a word on 3 different coordinates on your screen with 4 different functions and measures the time taken.  
Result:

    surface 	4.4340000022203e-05
    simple  	4.8489999971935e-05
    drawtext	5.563999984588e-05
    outlined	0.00012634444445009


As you can see above the `surface.*` method is the fastest closely followed by draw.SimpleText. It is (maybe) faster than `draw.DrawText` because it doesn't "handle newlines properly" according to the gmod wiki. SimpleTextOutlined is the slowest because it has to draw an extra outline in comparison to the 3 other functions.

The Code: [files/surface_vs_draw_text.lua](files/surface_vs_draw_text.lua)



# surface.DrawRect vs draw.RoundedBox

TL;DR: It is very slightly faster to use surface.DrawRect instead of draw.RoundedBox.  
It should not be important for performance though as draw.RoundedBox also supports round edges and is simpler to read and write.

    surface.DrawRect:   7.4859800010245e-06
    draw.RoundedBox:    8.335099999681e-06

The Code: [files/surface_vs_draw_box.lua](files/surface_vs_draw_box.lua)



# print vs Msg

TL;DR: MsgN and Msg are faster than `print`. Msg is only a bit faster than MsgN, because it doesn't append a newline at the end.

The test simply prints messages by using Msg, MsgN and print and checks the time taken.

Result:

    --- Benchmark complete
    reps	90	rounds	10
    On Server
    Msg	            1.9603333061645e-06
    print	        3.1246666549123e-06
    MsgN	        2.0377778006756e-06
    
    Msg vars	    2.9552222390016e-06
    MsgN vars	    2.9710000045371e-06
    print vars	    9.2549999974128e-06
    
    MsgN vars+tab	3.3575555587757e-06

The test is split into 2 sections: The first one where we only print a single string and the second one where we print varargs of 2 strings and 1 number.  
As you can see, print is always the slowest.  
The last benchmark is using MsgN with tabs to immitate what the print function does, and it is still faster than using print directly.

The Code: [files/print_vs_msgn.lua](files/print_vs_msgn.lua)



# Multiplication vs Division

TL;DR: In Garry's Mod it is not faster to calculate x * 0.5 than x / 2.
BUT: It is faster to multiply instead of divide in LUA5.3.

The result (average of 100 rounds with 1000000 calculations each):

    GMOD SERVERSIDE CONSOLE
    Multiplication:   0.00081622600000003
    Division:         0.00081775100000215
    LUA5.3 CONSOLE
    Multiplication:   0.01865942
    Division:         0.02342666

The Code: [files/multiplication_vs_division.lua](files/multiplication_vs_division.lua)



# Local vs Global variable speed

TL;DR: It is a little itsy-bitsy bit faster to make all variables local in Garry's Mod. (14% in this example)

The result (average of 100 rounds with 10000 calculations each):

    GMOD SERVERSIDE CONSOLE
    global math.sin:      0.0000059390000023996 (5.9390000023996e-06)
    local math.sin:       0.0000056840000024749 (5.6840000024749e-06)
    LUA5.3 CONSOLE
    global math.sin:      0.00057612
    local math.sin:       0.00049858

The Code: [files/local_vs_global.lua](files/local_vs_global.lua)



# x^2 vs x*x

This was tested with the inbuilt math.pow function and also the default power-of operator.

TL;DR: Using the "roof" operator is slightly faster in gmod. The speed is the same in Lua5.4.

The result (average of 100 rounds with 10000 calculations each):

    GMOD SERVERSIDE CONSOLE
    x^2 math.pow:  4.0210000111074e-06
    x*x:           3.7120000251889e-06
    x^2 ^:         3.5500000080901e-06
    LUA5.4 CONSOLE
    x^2 math.pow:  0.00015625
    x^2 ^:         0.00015625
    x*x:           0.00015625

The Code: [files/x_squared_vs_x_times.lua](files/x_squared_vs_x_times.lua)



# while vs for

TL;DR: A for loop is way faster than while.

The result (100x 10000):

    GMOD SERVERSIDE CONSOLE
    while:    0.00000957507999874 (9.57507999874e-06)
    for:      0.0000024306500009516 (2.4306500009516e-06)
    LUA5.3 CONSOLE
    while:    0.00015975869999999
    for:      0.000092178400000006 (9.2178400000006e-05)

The Code: [files/while_vs_for.lua](files/while_vs_for.lua)



# local Color() vs Color()

TL;DR: Defining the color once is faster than using Color() in a loop.

The result (1000x in SysTime() time taken):

    Local:     0.0000070583999997496 (7.0583999997496e-06)
    Non-Local: 0.0000091475999998067 (9.1475999998067e-06)

The Code: [files/local_color_vs_redefining.lua](files/local_color_vs_redefining.lua)



# Global variables slow down

TL;DR: Creating a lot of global variables neither slows down the game nor creates any noticable impact on performance.

I heard from multiple sources that "creating many global variables is bad".  
I tested this by creating 10.000 global variables and observing the game afterwards. An example with the runtime of 10.000 math function calls:

    Time    _G-count    time taken
    Before	3133        0.00044090000005781
    After	13144       0.00043160000018361

The FProfiler statistics and general gameplay feel didn't change either.  
For more details on the test (or if you want to verify it yourself) check out my script I used:  
[files/global_variables.lua](files/global_variables.lua)



# table.Count vs \#

TL;DR: It is better to always use the `#` operator instead of table.Count for sequential tables.

This comparison only works on sequential (or numerical) tables. These are tables that have a numerical, rising number as a key, in other languages you would call this "list" or array.

Table.count would always be used if you want to count a string-index table (also called dictionary or map), but in cases like this both can be used.

Result:

    # Table size: 1000
    Hashtag: 6.2949998141448e-08
    TbCount: 1.5430600009495e-06
    # Table size: 10
    Hashtag: 4.4980000029682e-08
    TbCount: 9.0410000939301e-08

As you can see above, the table.Count method is getting slower the larger the table becomes. This is because it loops through all the elements in the table and counts them, according to an old edit from the gmod wiki.

The Code: [files/table_count_vs_hashtag.lua](files/table_count_vs_hashtag.lua)



# ply.x vs tab[ply][x]

TL;DR: It is faster to get variables from a table than from the ply object. Setting a variable has nearly the same speed though, only getting the variable is slower on ply.

I and many others use code like `ply.lastSpawned = CurTime()`. This is quite easy to use and understand, but in performance aspects it is slower than using a table. An example with a table would be `tab[ply]["lastSpawned"] = CurTime()`.  
This is because reading this variable from a player uses a (slower) __index method. If you ever used FProfiler then you would probably have seen this "metamethod __index player" function near the top of "most runtime in ms".

I tested this by setting and getting random numbers with both methods 10.000 times, and I ran this 1000 times and calculated the average time taken. Result:

    ply:	0.0033284901999953
    tab:	0.0016247142000013

As you can see, it is quite faster (by 2x) with a table.  
Using a table also has its downsides, like having to clean it up regularly to avoid ram leaks and having to check 2 table indexes each time you want to verify it.  
Source code: [files/ply_vs_table_index.lua](files/ply_vs_table_index.lua)



# MySQL vs SQLite

For the full documentation please look at [https://shira.at/gmod/mysql_sqlite.html](https://shira.at/gmod/mysql_sqlite.html).  
Setup: MariaDB on Debian 10 installed on the same server as the Garry's Mod Server. Tested with DarkRP's MySQL functions and the mysqloo module.  

The time is measured from first requesting the data until we can use the data.  

Summary: MySQL is slower in comparison to SQLite. It needs an external connection and needs to wait for request/response delays either via network or via internal linux sockets. It's great for synchronisation of multiple servers, but it's not a "must-have" for servers.

## Query Delay Comparison

To get the delay for a query without read or write operation we can query `SELECT 1+1`.

Result:

    --First Test
    SQLite   ResponseTime:   0.000045883003622293s
    DRPMySQL ResponseTime:   0.020215623022523s
    --Second Test
    SQLite   ResponseTime:   0.000027909001801163s
    DRPMySQL ResponseTime:   0.049109992978629s

## Read and Write Comparison

Query: `SELECT rpname FROM darkrp_player WHERE uid = 76561198071737444`

Result:

    SQLite   Read Time:   0.00016302999938489s
    DRPMySQL Read Time:   0.048014674001024s

Query: `INSERT INTO darkrp_player VALUES(123123,'Test User',1,2)`

Result:

    SQLite   Write Time:   0.00020395400133566s
    DRPMySQL Write Time:   0.06270497100013s

## 1000 inserts (Bulk Inserts)

DarkRP's SQLite method uses sql.Begin and sql.Commit for SQLite based bulk imports. This makes it way faster than doing 1000 inserts after each other.  

DarkRP's MySQL method simply puts the queries into a table and sents them one after the other to the mysql-server. This is way slower, as the time neede is simply the time one insert takes times 1000.

Result:

    SQlite   1000 Inserts:     0.004849937000472s
    DRPMySQL 1000 Inserts:   ~30.000000000000000s (calculated)

