# postgres-influx-mimic
node proxy to query postgres but output as influx for grafana backend

Postgres db contains jsonb records looking like (the data could come from any table):

```
{
   "_id": "e832faf5b9421d53f7da348755db8e09",
   "_rev": "1-3b6906d2712b795411c1e0f3405f6d7c",
   "type": "counter",
   "name": "statsd.packets_received",
   "count": 7003,
   "ts": 1424734183
}
```

start the proxy 
```
mike@stuff.mw.office:~/pg-influx % node pg-influx.js
Listening on port 8086
Connected to postgres

```

in another window query it:
```
curl -G 'http://192.168.3.22:8086/query?pretty=true' --data-urlencode "q=WITH \
results AS ( \
  SELECT to_timestamp((doc->>'ts')::int) at time zone 'UTC' AS time, (doc->>'count')::numeric AS value FROM aatest \
  WHERE doc->>'name'='statsd.packets_received' AND (doc->>'count')::numeric > 0 \
  ORDER BY time \
), \
values AS (SELECT json_agg(json_build_array(time,value)) AS v FROM results) \
SELECT '{\"results\": [{ \
            \"series\": [{ \
                    \"name\": \"statsd.packets_received\", \
                    \"columns\": [\"time\", \"value\"], \
                    \"values\": ' || v || ' \
                }] \
        }]}' AS ret FROM values"
```

Should give back something similar to influx which can be used for grafana

```
[mike@f10 ~]$ curl -G 'http://192.168.3.22:8086/query?pretty=true' --data-urlencode "q=WITH \
> results AS ( \
>   SELECT to_timestamp((doc->>'ts')::int) at time zone 'UTC' AS time, (doc->>'count')::numeric AS value FROM aatest \
>   WHERE doc->>'name'='statsd.packets_received' AND (doc->>'count')::numeric > 0 \
>   ORDER BY time LIMIT 10\
> ), \
> values AS (SELECT json_agg(json_build_array(time,value)) AS v FROM results) \
> SELECT '{\"results\": [{ \
>             \"series\": [{ \
>                     \"name\": \"statsd.packets_received\", \
>                     \"columns\": [\"time\", \"value\"], \
>                     \"values\": ' || v || ' \
>                 }] \
>         }]}' AS ret FROM values"


{"results": [{             "series": [{
"name": "statsd.packets_received",   
"columns": ["time", "value"],
"values": [["2015-02-23T23:23:46", 1],
["2015-02-23T23:24:56", 501], 
["2015-02-23T23:25:46", 501], 
["2015-02-23T23:28:23", 501],
["2015-02-23T23:28:43", 89],
["2015-02-23T23:28:53", 3235], 
["2015-02-23T23:29:03", 1181],
["2015-02-23T23:29:33", 530],
["2015-02-23T23:29:43", 7003],
["2015-02-23T23:29:53", 5604]]                 }]         }]}
[mike@f10 ~]$ 

```

Getting results with different time format:
Change ```to_timestamp((doc->>'ts')::int) at time zone 'UTC' AS time```
To: ```(doc->>'ts')::numeric * 1000 AS time```
```
[1424789698000, 2712], 
[1424789701000, 1826],
[1424789704000, 263]
```

I have tested this and can see grafana making a request to the daemon:

```
/query?db=na&epoch=ms&p=na&q=WITH+results+AS+(+SELECT+(doc-%3E%3E%27ts%27)::numeric+*+1000+AS+time,+(doc-%3E%3E%27count%27)::numeric+AS+value+FROM+aatest+WHERE+doc-%3E%3E%27name%27%3D%27statsd.packets_received%27+AND+(doc-%3E%3E%27count%27)::numeric+%3E+0+ORDER+BY+time),+values+AS+(SELECT+json_agg(json_build_array(time,value))+AS+v+FROM+results)+SELECT+%27%7B%22results%22:+%5B%7B+%22series%22:+%5B%7B+%22name%22:+%22statsd.packets_received%22,+%22columns%22:+%5B%22time%22,+%22mean%22%5D,+%22values%22:+%27+%7C%7C+v+%7C%7C+%27+%7D%5D+%7D%5D%7D%27+AS+ret+FROM+values&u=na
```

however it doesnt seem to like the results returned current output from daemon looks like:

```
{"results": [{ "series": [{ "name": "statsd.packets_received", "columns": ["time", "value"], "values": [[1424733826000, 1], [1424733896000, 501], [1424733946000, 501], [1424734103000, 501], [1424734123000, 89], [1424734133000, 3235], [1424734143000, 1181], [1424734173000, 530], [1424734183000, 7003], [1424734193000, 5604], [1424734204000, 5558], [1424734214000, 6183], [1424734224000, 5865], [1424734234000, 5966], [1424734244000, 6005], [1424734254000, 5883], [1424734264000, 1205], [1424736339000, 1], [1424736372000, 1], [1424736445000, 1], [1424736553000, 3], [1424736681000, 4], [1424736892000, 1], [1424737072000, 1], [1424737092000, 1], [1424737316000, 1], [1424737514000, 1], [1424737549000, 1], [1424737726000, 1], [1424737808000, 1], [1424737838000, 1], [1424737894000, 3], [1424770719000, 1], [1424770747000, 1], [1424772606000, 2], [1424772616000, 2], [1424774591000, 1], [1424774674000, 3], [1424775160000, 3], [1424775644000, 1], [1424775689000, 1], [1424775722000, 1], [1424776635000, 1], [1424776908000, 1], [1424778046000, 2], [1424778076000, 153], [1424778086000, 51], [1424778217000, 69], [1424778227000, 162], [1424778252000, 163], [1424778262000, 30], [1424778312000, 58], [1424778322000, 161], [1424778333000, 54], [1424778363000, 117], [1424778373000, 258], [1424778383000, 147], [1424778413000, 124], [1424778423000, 250], [1424778663000, 122], [1424778673000, 88], [1424778693000, 201], [1424778703000, 930], [1424778713000, 686], [1424778767000, 636], [1424778777000, 939], [1424778787000, 477], [1424778867000, 927], [1424778877000, 944], [1424778903000, 954], [1424778913000, 365], [1424779083000, 497], [1424779093000, 947], [1424779103000, 947], [1424779113000, 921], [1424779123000, 943], [1424779133000, 946], [1424779143000, 941], [1424779153000, 930], [1424779163000, 947], [1424779173000, 945], [1424779183000, 944], [1424779193000, 943], [1424779203000, 935], [1424779213000, 941], [1424779223000, 935], [1424779233000, 956], [1424779243000, 911], [1424779253000, 927], [1424779263000, 645], [1424779273000, 368], [1424779284000, 958], [1424779294000, 949], [1424779304000, 959], [1424779314000, 942], [1424779324000, 955], [1424779334000, 957], [1424779344000, 953], [1424779354000, 954], [1424779364000, 977], [1424779374000, 957], [1424779384000, 958], [1424779394000, 935], [1424779405000, 956], [1424779415000, 937], [1424779425000, 948], [1424779435000, 954], [1424779445000, 951], [1424779455000, 954], [1424779465000, 956], [1424779475000, 268], [1424779485000, 651], [1424779495000, 956], [1424779505000, 962], [1424779516000, 966], [1424779526000, 316], [1424779536000, 616], [1424779546000, 958], [1424779556000, 962], [1424779566000, 963], [1424779577000, 573], [1424779587000, 384], [1424779597000, 1785], [1424779607000, 1863], [1424779617000, 1875], [1424779628000, 1740], [1424779648000, 1325], [1424779658000, 1931], [1424779669000, 1874], [1424779679000, 1868], [1424779689000, 1895], [1424779700000, 623], [1424779710000, 1107], [1424779720000, 1878], [1424779730000, 1909], [1424779741000, 1872], [1424779751000, 969], [1424779761000, 258], [1424779771000, 2682], [1424779781000, 2700], [1424779792000, 2705], [1424779802000, 2840], [1424779812000, 2995], [1424779823000, 2917], [1424779865000, 2620], [1424779875000, 2853], [1424779885000, 2797], [1424779895000, 2948], [1424779905000, 1170], [1424785839000, 14135], [1424785849000, 13950], [1424785980000, 18366], [1424785991000, 18893], [1424786002000, 19169], [1424786013000, 16325], [1424786024000, 18753], [1424786035000, 18569], [1424786076000, 18175], [1424786087000, 18834], [1424786099000, 19130], [1424786110000, 18157], [1424786122000, 10503], [1424786133000, 19109], [1424786145000, 25540], [1424786157000, 26891], [1424786169000, 26971], [1424786181000, 27859], [1424786192000, 32552], [1424786204000, 34537], [1424786558000, 4094], [1424786570000, 4495], [1424786581000, 4755], [1424786593000, 4237], [1424786605000, 4191], [1424786643000, 4605], [1424786655000, 4508], [1424786666000, 4827], [1424786679000, 142], [1424786719000, 4100], [1424786731000, 4229], [1424786776000, 7], [1424786786000, 4653], [1424786922000, 4588], [1424786933000, 5325], [1424787055000, 9158], [1424787067000, 9285], [1424787159000, 8990], [1424787170000, 9164], [1424787226000, 8900], [1424787238000, 8620], [1424787273000, 8802], [1424787284000, 7920], [1424787294000, 8561], [1424787304000, 7606], [1424787315000, 7814], [1424787344000, 13419], [1424787354000, 12358], [1424787365000, 12125], [1424787412000, 13419], [1424787423000, 12967], [1424787434000, 12880], [1424788386000, 1584], [1424788484000, 3036], [1424788598000, 1330], [1424788602000, 1488], [1424788606000, 1431], [1424788610000, 1358], [1424788615000, 1548], [1424788619000, 1176], [1424788623000, 1481], [1424788627000, 1293], [1424788631000, 1397], [1424788636000, 1401], [1424788663000, 1310], [1424788667000, 1459], [1424788671000, 1412], [1424788675000, 1218], [1424788679000, 1506], [1424788743000, 1105], [1424788746000, 1381], [1424788749000, 1456], [1424788753000, 1496], [1424788756000, 1425], [1424788759000, 1437], [1424788763000, 1437], [1424788766000, 1448], [1424788769000, 1481], [1424788822000, 3], [1424788826000, 3], [1424788829000, 3], [1424788832000, 3], [1424788835000, 2], [1424788856000, 1], [1424788888000, 1], [1424788906000, 3], [1424788909000, 3], [1424788912000, 3], [1424788915000, 3], [1424788933000, 5], [1424788936000, 2], [1424789057000, 51], [1424789060000, 96], [1424789063000, 97], [1424789066000, 37], [1424789072000, 238], [1424789075000, 942], [1424789078000, 947], [1424789081000, 946], [1424789084000, 940], [1424789098000, 947], [1424789101000, 965], [1424789104000, 956], [1424789107000, 955], [1424789120000, 140], [1424789129000, 1123], [1424789132000, 1408], [1424789135000, 1459], [1424789138000, 1404], [1424789141000, 524], [1424789144000, 1293], [1424789147000, 5297], [1424789150000, 5595], [1424789153000, 5497], [1424789156000, 5450], [1424789159000, 5592], [1424789162000, 5582], [1424789165000, 5636], [1424789168000, 6206], [1424789171000, 3884], [1424789174000, 1316], [1424789192000, 1154], [1424789195000, 1964], [1424789198000, 4893], [1424789201000, 6944], [1424789204000, 6988], [1424789207000, 6917], [1424789210000, 6967], [1424789213000, 6979], [1424789216000, 7365], [1424789219000, 8496], [1424789222000, 8837], [1424789226000, 10377], [1424789229000, 11193], [1424789232000, 12150], [1424789235000, 12585], [1424789238000, 12090], [1424789241000, 12234], [1424789244000, 12546], [1424789247000, 12568], [1424789251000, 12707], [1424789254000, 12828], [1424789257000, 13896], [1424789260000, 13812], [1424789285000, 45745], [1424789295000, 44995], [1424789306000, 45815], [1424789317000, 47681], [1424789327000, 49973], [1424789385000, 50504], [1424789396000, 37557], [1424789407000, 34848], [1424789439000, 10857], [1424789443000, 10131], [1424789446000, 5016], [1424789449000, 9174], [1424789453000, 10395], [1424789456000, 10131], [1424789459000, 9999], [1424789462000, 11550], [1424789465000, 12672], [1424789469000, 11979], [1424789472000, 12870], [1424789475000, 13761], [1424789478000, 13233], [1424789481000, 15178], [1424789484000, 15105], [1424789487000, 14781], [1424789491000, 14772], [1424789494000, 13540], [1424789497000, 13692], [1424789500000, 11954], [1424789503000, 12505], [1424789506000, 12429], [1424789509000, 12625], [1424789512000, 12581], [1424789516000, 12555], [1424789519000, 11626], [1424789522000, 10297], [1424789525000, 8895], [1424789528000, 8356], [1424789531000, 8366], [1424789534000, 8349], [1424789537000, 8448], [1424789540000, 8480], [1424789543000, 8530], [1424789546000, 8384], [1424789549000, 8322], [1424789552000, 8410], [1424789556000, 8501], [1424789559000, 8411], [1424789562000, 8476], [1424789565000, 8500], [1424789568000, 8484], [1424789571000, 7802], [1424789574000, 7105], [1424789577000, 5906], [1424789580000, 4415], [1424789583000, 4190], [1424789586000, 2892], [1424789589000, 2777], [1424789592000, 2774], [1424789595000, 2800], [1424789598000, 2749], [1424789601000, 2781], [1424789604000, 2745], [1424789607000, 2751], [1424789610000, 2794], [1424789613000, 2731], [1424789616000, 2801], [1424789619000, 2810], [1424789622000, 2779], [1424789625000, 2761], [1424789628000, 2759], [1424789631000, 2792], [1424789634000, 2788], [1424789637000, 2767], [1424789640000, 2789], [1424789644000, 2771], [1424789647000, 2786], [1424789650000, 2792], [1424789653000, 2767], [1424789656000, 2799], [1424789659000, 2776], [1424789662000, 2756], [1424789665000, 2730], [1424789668000, 2762], [1424789671000, 2785], [1424789674000, 2728], [1424789677000, 2745], [1424789680000, 2780], [1424789683000, 2711], [1424789686000, 2759], [1424789689000, 2770], [1424789692000, 2697], [1424789695000, 2753], [1424789698000, 2712], [1424789701000, 1826], [1424789704000, 263]] }] }]}
```
