---
layout: post
title: "CRC 체크섬 리버스 엔지니어링"
tags: [CRC, Checksum, Reverse Engineering]
comments: true
---

# CRC 체크섬 리버스 엔지니어링

이 글에서는 실제 통신 패킷을 분석하여 CRC알고리즘 모델을 역으로 산출하고
산출한 모델 파라미터를 기반으로 동작 가능한 CRC 소스코드를 작성하여 테스트한다. 

## CRC 리버스 엔지니어링 예시

아래와 같은 실제 패킷이 있다고 할 때 주의깊게 보면 어느정도의 규칙성을 찾을 수 있다. 

```plaintext

00000000:00
00010000:1A
00020000:01
00030000:1B
00040000:02
00050000:18
00060000:03
00070000:19
00080000:04
00090000:1E
000A0000:05
000B0000:1F
...(이하 생략)

```

위 패킷들의 경우 4바이트 페이로드에 1바이트 체크섬으로 구성되어 있는데, 체크섬의 값은 0x00-0x1F 로 구성되어 있다. 이것으로 미루어 보아 5비트 체크섬인 것으로 예상할 수 있다. 

통상 CRC체크섬의 경우 8의 배수 비트의 길이로 구성되기 때문에 이 경우는 꽤나 일반적이지 않은 예라고 할 수 있다. 

CRC 체크섬을 리버스 엔지니어링 할 수 있는 오픈소스 프로젝트는 [reveng](https://sourceforge.net/projects/reveng/) 에서 다운로드 받을 수 있다. 



## Reveng 사용법

`reveng.exe -h`를 통하여 프로그램의 사용법을 확인할 수 있다. 

```plaintext

CRC RevEng: arbitrary-precision CRC calculator and algorithm finder
Usage:  reveng.exe      -cdDesvhu? [-1bBfFGlLMrStVXyz]
        [-a BITS] [-A OBITS] [-i INIT] [-k KPOLY] [-m MODEL] [-p POLY]
        [-p POLY] [-P RPOLY] [-q QPOLY] [-w WIDTH] [-x XOROUT] [STRING...]
Options:
        -a BITS         bits per character (1 to 32)
        -A OBITS        bits per output character (1 to 32)
        -i INIT         initial register value
        -k KPOLY        generator in Koopman notation (implies WIDTH)
        -m MODEL        preset CRC algorithm
        -p POLY         generator or search range start polynomial
        -P RPOLY        reversed generator polynomial (implies WIDTH)
        -q QPOLY        search range end polynomial
        -w WIDTH        register size, in bits
        -x XOROUT       final register XOR value
Modifier switches:
        -1 skip equivalent forms        -b big-endian CRC
        -B big-endian CRC output        -f read files named in STRINGs
        -F skip preset model check pass -G skip brute force search pass
        -l little-endian CRC            -L little-endian CRC output
        -M non-augmenting algorithm     -r right-justified output
        -S print spaces between chars   -t left-justified output
        -V reverse algorithm only       -X print uppercase hexadecimal
        -y low bytes first in files     -z raw binary STRINGs
Mode switches:
        -c calculate CRCs               -d dump algorithm parameters
        -D list preset algorithms       -e echo (and reformat) input
        -s search for algorithm         -v calculate reversed CRCs
        -h | -u | -? show this help

Copyright (C) 2010, 2011, 2012, 2013, 2014,
              2015, 2016, 2017, 2018, 2019, 2020, 2021, 2022  Gregory Cook
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Version 3.0.5                             <https://reveng.sourceforge.io/>

```


### CRC 체크섬 모델 찾기

먼저 주어진 값 중 몇몇 값을 토대로 CRC값을 찾아보도록 하자. 
체크섬의 범위가 0x00-0x1F임을 알고 있으므로 CRC의 비트 수는 5개라고 예상할 수 있다. 

reveng 프로그램을 이용하여 CRC모델을 찾아본다. 
먼저 페이로드를 그대로 넣어 시험해본다.


```plaintext

D:\reveng\bin\win32>reveng.exe -w 5 -s 014F2F4006 014F2F4407 014F2F4804 014F2F4C05 014F2F5002 014F2F5403 014F2F5800 014F2F5C01 014F2F600E 014F2F640F 014F2F680C 014F2F6C0D 014F2F700A 014F2F740B 014F2F7808 014F2F7C09 014F2F8003 014F2F8402 014F2F8801 014F2F8C00 014F2F9007
reveng.exe: no models found

```

여기서 -w 옵션은 주어진 CRC의 비트 길이이며 -s 옵션은 주어진 값을 만족하는 CRC를 찾는다는 옵션이다. 

일부 값을 넣었을 때 해당 값을 기준으로 만족하는 CRC알고리즘이 없음을 알 수 있다. 

하지만 프로토콜에 따라 값을 Big endian으로 전송할 지 Little endian으로 전송할지가 다른 경우가 있다. 특히나 표준화되지 않은 통신인 경우 통신 패킷 구조에 따라 Big endian과 Little endian이 혼용되는 경우도 있다. 


다음으로는 32비트 페이로드의 순서를 뒤바꾸어 넣어본다(endianness 변경)

```plaintext

D:\reveng\bin\win32>reveng.exe -w 5 -s EC2E4F0107 F02E4F0100 F42E4F0101 F82E4F0102 FC2E4F0103 002F4F0116 042F4F0117 082F4F0114 0C2F4F0115 102F4F0112 142F4F0113 182F4F0110 1C2F4F0111 202F4F011E 242F4F011F 282F4F011C 2C2F4F011D 302F4F011A 342F4F011B 382F4F0118 3C2F4F0119 402F4F0106 442F4F0107 482F4F0104 4C2F4F0105 502F4F0102 542F4F0103 582F4F0100 5C2F4F0101 602F4F010E 642F4F010F 682F4F010C 6C2F4F010D 702F4F010A 742F4F010B 782F4F0108 7C2F4F0109 802F4F0103 842F4F0102 882F4F0101 8C2F4F0100 902F4F0107 942F4F0106 982F4F0105 9C2F4F0104 A02F4F010B A42F4F010A A82F4F0109 AC2F4F0108 B02F4F010F B42F4F010E B82F4F010D BC2F4F010C C02F4F0113 C42F4F0112 C82F4F0111 CC2F4F0110 D02F4F0117 D42F4F0116 D82F4F0115 DC2F4F0114 E02F4F011B E42F4F011A E82F4F0119 EC2F4F0118 F02F4F011F F42F4F011E F82F4F011D FC2F4F011C 00304F010A 04304F010B 08304F0108 0C304F0109 10304F010E 14304F010F 18304F010C 1C304F010D 20304F0102 24304F0103 28304F0100 2C304F0101 30304F0106 34304F0107 38304F0104 3C304F0105 40304F011A 44304F011B 48304F0118 4C304F0119 50304F011E 54304F011F 58304F011C 5C304F011D 60304F0112 64304F0113 68304F0110 6C304F0111 70304F0116 74304F0117 78304F0114 7C304F0115 80304F011F 84304F011E 88304F011D 8C304F011C 90304F011B 94304F011A 98304F0119 9C304F0118 A0304F0117 A4304F0116 A8304F0115 AC304F0114 B0304F0113 B4304F0112 B8304F0111 BC304F0110 C0304F010F C4304F010E C8304F010D CC304F010C D0304F010B D4304F010A D8304F0109 DC304F0108 E0304F0107 E4304F0106 E8304F0105 EC304F0104 F0304F0103 F4304F0102 F8304F0101 FC304F0100 00314F0115 04314F0114 08314F0117 0C314F0116 10314F0111 14314F0110 18314F0113 1C314F0112 20314F011D 24314F011C 28314F011F 2C314F011E 30314F0119 34314F0118 38314F011B 3C314F011A 40314F0105 44314F0104 48314F0107 4C314F0106 50314F0101 54314F0100 58314F0103 5C314F0102 60314F010D 64314F010C 68314F010F 6C314F010E 70314F0109 74314F0108 78314F010B 7C314F010A 80314F0100 84314F0101 88314F0102 8C314F0103 90314F0104 94314F0105 98314F0106 9C314F0107 A0314F0108 A4314F0109 A8314F010A AC314F010B B0314F010C B4314F010D B8314F010E BC314F010F C0314F0110 C4314F0111 C8314F0112 CC314F0113 D0314F0114 D4314F0115 D8314F0116 DC314F0117 E0314F0118 E4314F0119 E8314F011A EC314F011B F0314F011C F4314F011D F8314F011E FC314F011F 00324F0101 04324F0100 08324F0103 0C324F0102 10324F0105 14324F0104 18324F0107 1C324F0106 20324F0109 24324F0108 28324F010B 2C324F010A 30324F010D 34324F010C 38324F010F 3C324F010E 40324F0111 44324F0110 48324F0113 4C324F0112 50324F0115 54324F0114 58324F0117 5C324F0116 60324F0119 64324F0118 68324F011B 6C324F011A 70324F011D 74324F011C 78324F011F 7C324F011E 80324F0114 84324F0115 88324F0116 8C324F0117 90324F0110 94324F0111 98324F0112 9C324F0113 A0324F011C A4324F011D A8324F011E AC324F011F B0324F0118 B4324F0119 B8324F011A BC324F011B C0324F0104 C4324F0105 C8324F0106 CC324F0107 D0324F0100 D4324F0101 D8324F0102 DC324F0103 E0324F010C E4324F010D E8324F010E EC324F010F F0324F0108 F4324F0109 F8324F010A FC324F010B 00334F011E 04334F011F 08334F011C 0C334F011D 10334F011A 14334F011B 18334F0118 1C334F0119 20334F0116 24334F0117 28334F0114 2C334F0115 30334F0112 34334F0113 38334F0110 3C334F0111 40334F010E 44334F010F
width=5  poly=0x0b  init=0x00  refin=true  refout=true  xorout=0x00  check=0x02  residue=0x00  name=(none)

```

명령을 통해 아래와 같은 파라미터를 가지는 CRC 모델이 있음을 확인할 수 있다. 

- width=5  poly=0x0b  init=0x00  refin=true  refout=true  xorout=0x00  check=0x02  residue=0x00


### CRC 코드 생성

앞에서 언급한 것과 같이 5비트 CRC 체크섬은 대중적으로 사용하는 알고리즘은 아니기 때문에 효율화되지 않은 CRC 기본 알고리즘으로 동작하도록 코드를 만들어야 한다. 

인터넷을 찾으면 general한 방식으로 구현한 crc소스가 있어 해당 [소스코드](http://www.zorc.breitbandkatze.de/crctester.c)를 기반으로 위의 파라미터를 넣은 코드를 작성하였다. 

```c

// ----------------------------------------------------------------------------
// CRC tester v1.3 written on 4th of February 2003 by Sven Reifegerste (zorc/reflex)
// This is the complete compilable C program, consisting only of this .c file.
// No guarantee for any mistakes.
//
// changes to CRC tester v1.2:
//
// - remove unneccessary (!(polynom&1)) test for invalid polynoms
//   (now also XMODEM parameters 0x8408 work in c-code as they should)
//
// changes to CRC tester v1.1:
//
// - include an crc&0crcmask after converting non-direct to direct initial
//   value to avoid overflow
//
// changes to CRC tester v1.0:
//
// - most int's were replaced by unsigned long's to allow longer input strings
//   and avoid overflows and unnecessary type-casting's
// ----------------------------------------------------------------------------
 
// includes:
 
#include <string.h>
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>
 
// CRC parameters (default values are for CRC-32):
 
const int order = 5;
const unsigned long polynom = 0x0b;
const int direct = 1;
const unsigned long crcinit = 0x0;
const unsigned long crcxor = 0x0;
const int refin = 1;
const int refout = 1;
 
// 'order' [1..32] is the CRC polynom order, counted without the leading '1' bit
// 'polynom' is the CRC polynom without leading '1' bit
// 'direct' [0,1] specifies the kind of algorithm: 1=direct, no augmented zero bits
// 'crcinit' is the initial CRC value belonging to that algorithm
// 'crcxor' is the final XOR value
// 'refin' [0,1] specifies if a data byte is reflected before processing (UART) or not
// 'refout' [0,1] specifies if the CRC will be reflected before XOR
 
 
// Data character string
 
unsigned char string[] = {"123456789"};
char string_length = 0;
// internal global values:
 
unsigned long crcmask;
unsigned long crchighbit;
unsigned long crcinit_direct;
unsigned long crcinit_nondirect;
unsigned long crctab[256];
 
 
// subroutines
 
unsigned long reflect (unsigned long crc, int bitnum) {
 
    // reflects the lower 'bitnum' bits of 'crc'
 
    unsigned long i, j=1, crcout=0;
 
    for (i=(unsigned long)1<<(bitnum-1); i; i>>=1) {
        if (crc & i) crcout|=j;
        j<<= 1;
    }
    return (crcout);
}
 
 
 
void generate_crc_table() {
 
    // make CRC lookup table used by table algorithms
 
    int i, j;
    unsigned long bit, crc;
 
    for (i=0; i<256; i++) {
 
        crc=(unsigned long)i;
        if (refin) crc=reflect(crc, 8);
        crc<<= order-8;
 
        for (j=0; j<8; j++) {
 
            bit = crc & crchighbit;
            crc<<= 1;
            if (bit) crc^= polynom;
        }            
 
        if (refin) crc = reflect(crc, order);
        crc&= crcmask;
        crctab[i]= crc;
    }
}
 
 
        
unsigned long crctablefast (unsigned char* p, unsigned long len) {
 
    // fast lookup table algorithm without augmented zero bytes, e.g. used in pkzip.
    // only usable with polynom orders of 8, 16, 24 or 32.
 
    unsigned long crc = crcinit_direct;
 
    if (refin) crc = reflect(crc, order);
 
    if (!refin) while (len--) crc = (crc << 8) ^ crctab[ ((crc >> (order-8)) & 0xff) ^ *p++];
    else while (len--) crc = (crc >> 8) ^ crctab[ (crc & 0xff) ^ *p++];
 
    if (refout^refin) crc = reflect(crc, order);
    crc^= crcxor;
    crc&= crcmask;
 
    return(crc);
}
 
 
 
unsigned long crctable (unsigned char* p, unsigned long len) {
 
    // normal lookup table algorithm with augmented zero bytes.
    // only usable with polynom orders of 8, 16, 24 or 32.
 
    unsigned long crc = crcinit_nondirect;
 
    if (refin) crc = reflect(crc, order);
 
    if (!refin) while (len--) crc = ((crc << 8) | *p++) ^ crctab[ (crc >> (order-8))  & 0xff];
    else while (len--) crc = ((crc >> 8) | (*p++ << (order-8))) ^ crctab[ crc & 0xff];
 
    if (!refin) while (++len < order/8) crc = (crc << 8) ^ crctab[ (crc >> (order-8))  & 0xff];
    else while (++len < order/8) crc = (crc >> 8) ^ crctab[crc & 0xff];
 
    if (refout^refin) crc = reflect(crc, order);
    crc^= crcxor;
    crc&= crcmask;
 
    return(crc);
}
 
 
 
unsigned long crcbitbybit(unsigned char* p, unsigned long len) {
 
    // bit by bit algorithm with augmented zero bytes.
    // does not use lookup table, suited for polynom orders between 1...32.
 
    unsigned long i, j, c, bit;
    unsigned long crc = crcinit_nondirect;
 
    for (i=0; i<len; i++) {
 
        c = (unsigned long)*p++;
        if (refin) c = reflect(c, 8);
 
        for (j=0x80; j; j>>=1) {
 
            bit = crc & crchighbit;
            crc<<= 1;
            if (c & j) crc|= 1;
            if (bit) crc^= polynom;
        }
    }    
 
    for (i=0; i<order; i++) {
 
        bit = crc & crchighbit;
        crc<<= 1;
        if (bit) crc^= polynom;
    }
 
    if (refout) crc=reflect(crc, order);
    crc^= crcxor;
    crc&= crcmask;
 
    return(crc);
}
 
 
 
unsigned long crcbitbybitfast(unsigned char* p, unsigned long len) {
 
    // fast bit by bit algorithm without augmented zero bytes.
    // does not use lookup table, suited for polynom orders between 1...32.
 
    unsigned long i, j, c, bit;
    unsigned long crc = crcinit_direct;
 
    for (i=0; i<len; i++) {
 
        c = (unsigned long)*p++;
        if (refin) c = reflect(c, 8);
 
        for (j=0x80; j; j>>=1) {
 
            bit = crc & crchighbit;
            crc<<= 1;
            if (c & j) bit^= crchighbit;
            if (bit) crc^= polynom;
        }
    }    
 
    if (refout) crc=reflect(crc, order);
    crc^= crcxor;
    crc&= crcmask;
 
    return(crc);
}
 
 
 
int main(int argc, char* argv[]) {
 
    // test program for checking four different CRC computing types that are:
    // crcbit(), crcbitfast(), crctable() and crctablefast(), see above.
    // parameters are at the top of this program.
    // Result will be printed on the console.
 
    int i;
    unsigned long bit, crc;
 
 
    // at first, compute constant bit masks for whole CRC and CRC high bit
 
    crcmask = ((((unsigned long)1<<(order-1))-1)<<1)|1;
    crchighbit = (unsigned long)1<<(order-1);
 
 
    // check parameters
 
    if (order < 1 || order > 32) {
        printf("ERROR, invalid order, it must be between 1..32.\n");
        return(0);
    }
 
    if (polynom != (polynom & crcmask)) {
        printf("ERROR, invalid polynom.\n");
        return(0);
    }
 
    if (crcinit != (crcinit & crcmask)) {
        printf("ERROR, invalid crcinit.\n");
        return(0);
    }
 
    if (crcxor != (crcxor & crcmask)) {
        printf("ERROR, invalid crcxor.\n");
        return(0);
    }
 
    
    // generate lookup table
 
    generate_crc_table();
 
 
    // compute missing initial CRC value
 
    if (!direct) {
 
        crcinit_nondirect = crcinit;
        crc = crcinit;
        for (i=0; i<order; i++) {
 
            bit = crc & crchighbit;
            crc<<= 1;
            if (bit) crc^= polynom;
        }
        crc&= crcmask;
        crcinit_direct = crc;
    }
 
    else {
 
        crcinit_direct = crcinit;
        crc = crcinit;
        for (i=0; i<order; i++) {
 
            bit = crc & 1;
            if (bit) crc^= polynom;
            crc >>= 1;
            if (bit) crc|= crchighbit;
        }    
        crcinit_nondirect = crc;
    }
 
 
    /* User Control Area */
    if(2 < argc){
        uint32_t tmp = strtoul(argv[1],NULL, 16);
        uint32_t crc = strtoul(argv[2],NULL, 16);
        string_length = strlen(argv[1])/2 + strlen(argv[2])/2;
 
        // printf("tmp : %08x\n", tmp);
        // printf("crc : %08x\n", crc);
        // printf("string_length = %d\n", string_length);
 
        memcpy((void *)&string[0], (void *)&tmp, sizeof(tmp));
        memcpy((void *)&string[4], (void *)&crc, sizeof(crc));
        // for(int i = 0; i <  5; i++){
        //     printf("%02x ", string[i]);
        // }
 
    }
 
 
 
    // END    
 
    // call CRC algorithms using the CRC parameters above and print result to the console
 
    // printf("\n");
    // printf("CRC tester v1.1 written on 13/01/2003 by Sven Reifegerste (zorc/reflex)\n");
    // printf("-----------------------------------------------------------------------\n");
    // printf("\n");
    // printf("Parameters:\n");
    // printf("\n");
    // printf(" polynom             :  0x%x\n", polynom);
    // printf(" order               :  %d\n", order);
    // printf(" crcinit             :  0x%x direct, 0x%x nondirect\n", crcinit_direct, crcinit_nondirect);
    // printf(" crcxor              :  0x%x\n", crcxor);
    // printf(" refin               :  %d\n", refin);
    // printf(" refout              :  %d\n", refout);
    // printf("\n");
    // printf(" data string         :  '%s' (%d bytes)\n", string, string_length);
    // printf("\n");
    // printf("Results:\n");
    // printf("\n");
 
    // printf(" crc bit by bit      :  0x%x\n", crcbitbybit((unsigned char *)string, string_length));
    // printf(" crc bit by bit fast :  0x%x\n", crcbitbybitfast((unsigned char *)string, string_length));
    // if (!(order&7)) printf(" crc table           :  0x%x\n", crctable((unsigned char *)string, string_length));
    // if (!(order&7)) printf(" crc table fast      :  0x%x\n", crctablefast((unsigned char *)string, string_length));
 
    printf("0x%02x\n", crcbitbybit((unsigned char *)string, string_length));
    return(0);
}

```


위의 소스코드를 통하여 주어진 패킷에 대해 CRC가 정상적으로 작성되는 것을 확인할 수 있으며 CRC가 포함된 전체 패킷을 전송할 때 나머지가 0으로 정상적인 validation이 이루어 질 수 있음을 알 수 있다. 

상세한 자료는 [해당 레포지토리](https://github.com/naudhizb/crc_reverse_engineering)를 참고해보자. 