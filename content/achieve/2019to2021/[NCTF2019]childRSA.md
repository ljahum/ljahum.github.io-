---
title: "[NCTF2019]childRSA"
date: 2020-10-02 01:40:43
tags:
- crypto

categories:
- CTF

math:
  enable: true

---
> 今天碰巧又做了一遍这个题,除了之前用yafu嗯解，又去参考了一下paper解法

# 	Challenge Code

```python
from random import choice
from Crypto.Util.number import isPrime, sieve_base as primes
from flag import flag


def getPrime(bits):
    while True:
        n = 2
        while n.bit_length() < bits:
            n *= choice(primes) # 前10000个速=素数中随机选一个
        if isPrime(n + 1):  
            return n + 1

e = 0x10001
m = int.from_bytes(flag.encode(), 'big')
p, q = [getPrime(2048) for _ in range(2)]    # q = 素数*素数....*素数 + 1
n = p * q
c = pow(m, e, n)

# n = 32849718197337581823002243717057659218502519004386996660885100592872201948834155543125924395614928962750579667346279456710633774501407292473006312537723894221717638059058796679686953564471994009285384798450493756900459225040360430847240975678450171551048783818642467506711424027848778367427338647282428667393241157151675410661015044633282064056800913282016363415202171926089293431012379261585078566301060173689328363696699811123592090204578098276704877408688525618732848817623879899628629300385790344366046641825507767709276622692835393219811283244303899850483748651722336996164724553364097066493953127153066970594638491950199605713033004684970381605908909693802373826516622872100822213645899846325022476318425889580091613323747640467299866189070780620292627043349618839126919699862580579994887507733838561768581933029077488033326056066378869170169389819542928899483936705521710423905128732013121538495096959944889076705471928490092476616709838980562233255542325528398956185421193665359897664110835645928646616337700617883946369110702443135980068553511927115723157704586595844927607636003501038871748639417378062348085980873502535098755568810971926925447913858894180171498580131088992227637341857123607600275137768132347158657063692388249513
# c = 26308018356739853895382240109968894175166731283702927002165268998773708335216338997058314157717147131083296551313334042509806229853341488461087009955203854253313827608275460592785607739091992591431080342664081962030557042784864074533380701014585315663218783130162376176094773010478159362434331787279303302718098735574605469803801873109982473258207444342330633191849040553550708886593340770753064322410889048135425025715982196600650740987076486540674090923181664281515197679745907830107684777248532278645343716263686014941081417914622724906314960249945105011301731247324601620886782967217339340393853616450077105125391982689986178342417223392217085276465471102737594719932347242482670320801063191869471318313514407997326350065187904154229557706351355052446027159972546737213451422978211055778164578782156428466626894026103053360431281644645515155471301826844754338802352846095293421718249819728205538534652212984831283642472071669494851823123552827380737798609829706225744376667082534026874483482483127491533474306552210039386256062116345785870668331513725792053302188276682550672663353937781055621860101624242216671635824311412793495965628876036344731733142759495348248970313655381407241457118743532311394697763283681852908564387282605279108

```


## 解法1

用yafu强行分解因数，因为数据太长，必须放入文件中来读取

```shell
16953 :: ~\Desktop\Daily\oi\crypto » yafu "factor(@)" -batchfile .\data.txt
.
.
***factors found***

PRP621 = 178449493212694205742332078583256205058672290603652616240227340638730811945224947826121772642204629335108873832781921390308501763661154638696935732709724016546955977529088135995838497476350749621442719690722226913635772410880516639651363626821442456779009699333452616953193799328647446968707045304702547915799734431818800374360377292309248361548868909066895474518333089446581763425755389837072166970684877011663234978631869703859541876049132713490090720408351108387971577438951727337962368478059295446047962510687695047494480605473377173021467764495541590394732685140829152761532035790187269724703444386838656193674253139
PRP621 = 184084121540115307597161367011014142898823526027674354555037785878481711602257307508985022577801782788769786800015984410443717799994642236194840684557538917849420967360121509675348296203886340264385224150964642958965438801864306187503790100281099130863977710204660546799128755418521327290719635075221585824217487386227004673527292281536221958961760681032293340099395863194031788435142296085219594866635192464353365034089592414809332183882423461536123972873871477755949082223830049594561329457349537703926325152949582123419049073013144325689632055433283354999265193117288252918515308767016885678802217366700376654365502867

ans = 1
```

几秒就跑出来了，应该是程序察觉出了 n 可以用算法快速分解

## 解法二

预期解为`Pollard' p-1`算法和`smooth number`两个知识点

先介绍知识点

### Pollard' p-1 算法

[https://blog.csdn.net/weixin_42251364/article/details/95462358]()

pollard’s p-1方法有点特殊，它只能应用在求整数n的一个素因子p，且p-1能被“小”因子整除的情况下，除此之外该方法无法正常应用。但是这个方法运用起来相当简单，所以在防止因式分解攻击时，必须考虑这一方法。

考虑在一个标准的rsa中

$n=p*q$

设  $x\equiv1\;mod\;p$

故有 p|gcd(x-1,n)

- 考虑费马小定理，有$a^{p-1}\equiv1\;mod\;p$

现在我们考虑构造一个$p-1$的倍数

- 若$p-1$有多个**小素因子**，且每一个素数幂$q|(p-1)$，存在一个数B使$q<=B$
- 则计算$B!$的值即为$p-1$的倍数

考虑费马小定理，则有$a^{B!}\equiv\;a^{k*(p-1)}\equiv1\;mod\;p$

$a^{B!}\equiv\;a^{k*(p-1)}\equiv\;x\;mod\;n$

模除的分配律：(参考wiki模运算性质)
$$
d\;mod\;abc\equiv\;(d\;mod\;a)+a*[(d/a)mod\;b]+ab[(d/ a/b)]\;mod \;c
$$

故$x=\;(a^{k*(p-1)}mod\;p)+p[(x/p)\;mod\;q]=1+k*p$

故该$x-1$为$p$的倍数

故计算$q=gcd(x-1),n=gcd(k*p,q*p)$

得到q后可以轻松得到flag


### exp

```python
# python2
from Crypto.Util.number import sieve_base as primes
import gmpy2
n = 32849718197337581823002243717057659218502519004386996660885100592872201948834155543125924395614928962750579667346279456710633774501407292473006312537723894221717638059058796679686953564471994009285384798450493756900459225040360430847240975678450171551048783818642467506711424027848778367427338647282428667393241157151675410661015044633282064056800913282016363415202171926089293431012379261585078566301060173689328363696699811123592090204578098276704877408688525618732848817623879899628629300385790344366046641825507767709276622692835393219811283244303899850483748651722336996164724553364097066493953127153066970594638491950199605713033004684970381605908909693802373826516622872100822213645899846325022476318425889580091613323747640467299866189070780620292627043349618839126919699862580579994887507733838561768581933029077488033326056066378869170169389819542928899483936705521710423905128732013121538495096959944889076705471928490092476616709838980562233255542325528398956185421193665359897664110835645928646616337700617883946369110702443135980068553511927115723157704586595844927607636003501038871748639417378062348085980873502535098755568810971926925447913858894180171498580131088992227637341857123607600275137768132347158657063692388249513
c = 26308018356739853895382240109968894175166731283702927002165268998773708335216338997058314157717147131083296551313334042509806229853341488461087009955203854253313827608275460592785607739091992591431080342664081962030557042784864074533380701014585315663218783130162376176094773010478159362434331787279303302718098735574605469803801873109982473258207444342330633191849040553550708886593340770753064322410889048135425025715982196600650740987076486540674090923181664281515197679745907830107684777248532278645343716263686014941081417914622724906314960249945105011301731247324601620886782967217339340393853616450077105125391982689986178342417223392217085276465471102737594719932347242482670320801063191869471318313514407997326350065187904154229557706351355052446027159972546737213451422978211055778164578782156428466626894026103053360431281644645515155471301826844754338802352846095293421718249819728205538534652212984831283642472071669494851823123552827380737798609829706225744376667082534026874483482483127491533474306552210039386256062116345785870668331513725792053302188276682550672663353937781055621860101624242216671635824311412793495965628876036344731733142759495348248970313655381407241457118743532311394697763283681852908564387282605279108
t=pow(2,2048)
e = 0x10001
k=2
for i in range(10000):
	k=pow(k,primes[i],n)  # 没乘方一次模一次n和最后再来模n的结果是一样的
	if(k>t):
		if(i%15==0):     #   随心情定每几次后判断一次 也可以乘方10000次后再判断，就怕电脑遭不住
 			if(gmpy2.gcd(k-1,n)!=1):  # n只有q、p两个因数
				print(gmpy2.gcd(k-1,n))
				#178449493212694205742332078583256205058672290603652616240227340638730811945224947826121772642204629335108873832781921390308501763661154638696935732709724016546955977529088135995838497476350749621442719690722226913635772410880516639651363626821442456779009699333452616953193799328647446968707045304702547915799734431818800374360377292309248361548868909066895474518333089446581763425755389837072166970684877011663234978631869703859541876049132713490090720408351108387971577438951727337962368478059295446047962510687695047494480605473377173021467764495541590394732685140829152761532035790187269724703444386838656193674253139
				break
p=gmpy2.gcd(k-1,n)
q=n//p
phi=(p-1)*(q-1)
d=gmpy2.invert(e,phi)
m=hex(pow(c,d,n))[2:]
print(bytes.fromhex(m).decode('utf-8'))

#NCTF{Th3r3_ar3_1ns3cure_RSA_m0duli_7hat_at_f1rst_gl4nce_appe4r_t0_be_s3cur3}

```












