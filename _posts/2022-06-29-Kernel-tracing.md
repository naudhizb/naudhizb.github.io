---
layout: post
title: "리눅스 커널 추적 및 분석"
tags: [Linux, Kernel, tracing, uftrace]
comments: true
---

# Linux 중급과정

 기존 리눅스 이론 기반의 과정이나, 구버전 리눅스 커널 기반의 교육에서 벗어나, 최신 커널 버전에서 트레이싱 기술을 접목하여 핵심적이고 실용적인 교육을 지향

## 강의 내용

0. 커널 트레이싱 방법 및 실습
    - uftrace
    - 커널 트레이싱
1. 메모리 영역 및 메모리 변환과정
    - 메모리 => 가상메모리 => user / kernel
    - 물리 메모리(Physical Memory)와 가상 메모리(Virtual Memory)를 변환하는 과정
    - PageFault 발생 시 커널의 코드 흐름
    - User와 Kernel의 PageFault 핸들링 코드의 흐름
2. 파일 시스템 I/O => Buffered I/O
3. 블록 디바이스 처리
4. 스케쥴러 / 시그널
5. 네트워크 / 인터럽트 / 후반부 처리(softirq, tasklet, workqueue)

## 공부 방법

리눅스 커널은 규모가 매우 큰 오픈소스 프로젝트이기 때문에 모든 내용을 파악하는것은 매우 어렵고 오래 걸린다. 
따라서 전체적인 내용을 파악한 뒤에 점점 깊이를 확장시켜가는 것이 효율적이다. 
가령 depth가 1부터 10까지 있다면 처음에는 depth를 4 또는 5정도로 해서 전체적인 부분을 이해할 수 있도록 한뒤에 depth를 늘려가는 것이 좋다. 

# 1일차

## 리눅스 OS 역할과 구성

### OS가 하는 일은? 

1. 사용자 응용 관리
2. 하드웨어 자원 관리

### OS의 구성

1. Core 부분
 - PM: process management
	* task
	* schedular
	* ...
 - MM: memory management
	* virtual memory
	* paging
 - IRQ / exception 처리, locking
	* Interrupt
	* Page Fault
	* Mutex
2. I/O 처리
 - 네트워크
	* L4: TCP, ...
	* L3: IP
	* L2: DD
 - 스토리지
	* VFS
	* FS
	* Block
 - 디바이스 드라이버
	* 개발이 가장 활발한 부분
3. 기타
    * security
    * tools
    * sounds
    * ...

###### 실습 1. Linux 커널 Commit 분석: 어느 파트가 제일 활발하게 개발될까? 

리눅스 커널 코드에서 어떤 개발자가 제일 많이 기여했을까? 
```
cd ~/git/linux
git shortlog -sn --no-merges | nl | less
```

리눅스 커널 코드에서 제일 많이 기여한 개발자는 어떤파트 담당일까? 

```
# 리눅스 커널 소스코드 폴더로 이동
$ cd ~/git/linux/

# 리눅스 커널 내부 서브 프로젝트 확인하기  
$ vim MAINTAINERS
```

폴더에서 특정 기간 사이에 해당 파일 관련한 커밋 내역 보기
```
reallinux@ubuntu:~/git/linux/net/core$ git log --oneline --after=2018-01-01 --before=2018-12-31 -- filter.c
7a69c0f25056 bpf: skmsg, replace comments with BUILD bug
bc1b4f013b50 bpf: sk_msg, improve offset chk in _is_valid_access
3bdbd0228e75 bpf: sockmap, metadata support for reporting size of msg
...
```

2018년도에 net/core 영역에서는 bpf라는 기능이 hot 했다. 
Add, Implement, Support ... : 개발, 추가개발 등
Fix, Improve, Refactor, Remove, ... : 유지보수
=> 커널에서 Commit log를 기록하는 기준이 있고, 해당 내용을 보면 그 내용을 예측할 수 있다. 

###### 실습 2. 리눅스 커널도 C프로그램이다. readelf, objdump 실습

hello world 프로그램을 작성하고 readelf 프로그램을 통하여 .text섹션에서 main 함수를 찾기

```
readelf -l a.out
```


리눅스 커널 바이너리인 vmlinux에서 .text섹션을 보고 start_kernel 함수를 찾아보자.
```
readelf -l vmlinux
objdump -d vmlinux | less

reallinux@ubuntu:~/git/linux$ readelf -l vmlinux

Elf file type is EXEC (Executable file)
Entry point 0x1000000
There are 5 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000200000 0xffffffff81000000 0x0000000001000000
                 0x0000000001320000 0x0000000001320000  R E    0x200000
  LOAD           0x0000000001600000 0xffffffff82400000 0x0000000002400000
                 0x00000000005af000 0x00000000005af000  RW     0x200000
  LOAD           0x0000000001c00000 0x0000000000000000 0x00000000029af000
                 0x000000000002a558 0x000000000002a558  RW     0x200000
  LOAD           0x0000000001dda000 0xffffffff829da000 0x00000000029da000
                 0x0000000000186000 0x0000000000252000  RWE    0x200000
  NOTE           0x00000000010010d4 0xffffffff81e010d4 0x0000000001e010d4
                 0x000000000000003c 0x000000000000003c         0x4

 Section to Segment mapping:
  Segment Sections...
   00     .text .notes __ex_table .rodata .pci_fixup .tracedata __ksymtab __ksymtab_gpl __ksymtab_strings __param __modver
   01     .data __bug_table .orc_unwind_ip .orc_unwind .orc_lookup .vvar
   02     .data..percpu
   03     .init.text .altinstr_aux .init.data .x86_cpu_dev.init .altinstructions .altinstr_replacement .iommu_table .apicdrivers .exit.text .smp_locks .data_nosave .bss .brk
   04     .notes

```


###### 실습 3. 리눅스 커널함수가 호출되는 과정 추적. ctags, uftrace 실습

CPU의 실행의 단위는 명령어(Instruction)이다. 
하지만, 사용자의 관점에서는 함수 단위로 실행하게 된다.
함수의 관점으로 보았을 때 컴퓨터에서 실행하는 함수의 종류는 3가지이다. 

- 유저함수
- 라이브러리 함수
- 커널 함수(커널 함수는 어떤 경우에 실행될까? 커널 함수를 실행하도록 하는 진입점(Entry)는)
    * 예외처리(e.g. System call, Page fault, ...)
    * 인터럽트(e.g. Connection USB, Network Packet Arrival, ...)
    * 커널 태스크(진입점은 아님)(e.g. 후반부 작업처리: softirq, tasklet, workqueue, ...) 
        => 나중에 호출해도 되는 함수들을 미룬다. 
        => 커널 태스크를 정상적으로 실행되지 않도록 공격하는 것이 DoS(Denial Of Service) 공격
        
시스템 콜: OS, 컴파일러, CPU가 함께 약속 => 규약(ABI) 들 중 한개
=> 해당 규약을 통해서 재컴파일을 줄일 수 있음. 

시스템콜을 사용할 때
- x86: syscall instruction
    * rax register(절반만 사용 시 eax)
- ARM: SVN intruction
    * x8 레지스터
    
path name(경로명) => inode
name to inode => namei.c

** ctags 사용하기 **

```
$sudo apt-get install ctags
cd ${DEV_PATH}
ctags -R
vi ~/.vimrc
 > set tags = ${DEV_PATH} 추가
```

- ^ + ] : 해당 심볼로 점프
- ^ + t : 이전 심볼로 복귀
- :tj [SYMBOL] : 해당 심볼 후보 리스팅
- :e [FILENAME] : 파일 열기

** uftrace 사용하기 **

```
touch reallinux
sudo uftrace record --force -K 5 /bin/mv reallinux reallinux-jjang
uftrace replay -t 8us
ldd /bin/mv
> mv프로그램의 rename은 어떤 라이브러리에 구현되어있을까? 
objdump -d /lib/x86_64-linux-gnu/libc.so.6 | less
> /rename
```


###### 실습 4. 커널함수(시스템콜함수) 처리과정 추적

리눅스 커널 => vmlinux 라는 C 프로그램 => 수천개의 함수들이 있음. 

=> 몇 개의 함수는 user application 에게 호출할 수 있게 제공한다. 
==> 시스템 콜 : 주로 사용하는 시스템콜 open, read, write

```
cd ${KERNEL_PATH}
vi arch/x86/entry/entry_64.S
> /do_syscall_64 --> 함수 내용을 찾아보자.
vi arch/x86/entry/common.c
> /do_syscall_64 --> sys_call_table[]에서 82번에 rename 함수 포인터가 세팅되어있다.
ag SYSCALL_DEFINE | grep rename
ag SYSCALL_DEFINE | grep open
ag SYSCALL_DEFINE | grep read
ag SYSCALL_DEFINE | grep write
vi fs/namei.c
> /rename
ag ext4_rename2
```

## Memory management

vmlinux(가상 메모리를 사용하는 리눅스) 의 경우 주소를 이용한 동작 시에 PA가 아닌 VA를 기반으로 동작 시 VA를 PA로 변환하여 동작한다. 
PA를 이용하는 것이 구조적으로 간단하나, 다른 실행파일들과의 충돌 가능성과 자원 관리 그리고 유지 보수성 면에서 VA를 사용하는 것이 더욱 효율적이기 때문에 VA를 사용한다. 


### Page Fault는 왜 발생할까? 

VA와 PA와의 변환 예시 (VSZ: 가상 메모리 공간, RSS: 물리 메모리 공간)

```
reallinux@ubuntu:~/git/linux/fs$ ps -eo pid,comm,vsz,rss | head -2
  PID COMMAND            VSZ   RSS
    1 systemd          77676  8852
reallinux@ubuntu:~/git/linux/fs$ sudo cat /proc/1/maps | grep systemd
555b2e35f000-555b2e4ad000 r-xp 00000000 08:02 935045                     /lib/systemd/systemd
555b2e6ac000-555b2e6e4000 r--p 0014d000 08:02 935045                     /lib/systemd/systemd
555b2e6e4000-555b2e6e5000 rw-p 00185000 08:02 935045                     /lib/systemd/systemd
7fdff8206000-7fdff83bc000 r-xp 00000000 08:02 918239                     /lib/systemd/libsystemd-shared-237.so
7fdff83bc000-7fdff85bc000 ---p 001b6000 08:02 918239                     /lib/systemd/libsystemd-shared-237.so
7fdff85bc000-7fdff8646000 r--p 001b6000 08:02 918239                     /lib/systemd/libsystemd-shared-237.so
7fdff8646000-7fdff8647000 rw-p 00240000 08:02 918239                     /lib/systemd/libsystemd-shared-237.so
reallinux@ubuntu:~/git/linux/fs$ sudo cat /proc/1/maps | grep heap
555b2e90f000-555b2ea73000 rw-p 00000000 00:00 0                          [heap]
reallinux@ubuntu:~/git/linux/fs$ sudo cat /proc/1/maps | grep stack
7ffed8e8c000-7ffed8ead000 rw-p 00000000 00:00 0                          [stack]
> r: readable(읽기), x: executable(실행), p: private(not shared)
```


# 2일차

### 가상 메모리와 실제 메모리가 변환되기 위한 과정

(1) int a = 1; 텍스트코드 컴파일
(2) mov 1 addr(VA);
(3) 프로세스 실행 fork() => task_struct 생성
(4) fork() 시스템 콜 => task_struct - mm_struct -> vma(stack), vma(heap), vma(code), ...
(5) 현재 프로세스를 위한 VA-PA 변환테이블(페이지테이블) 준비(pgd) 
(5-1) runqueue 삽입 => 나중에 스케쥴러가 선택하여 실행
(5-2) 현재 페이지 테이블의 첫주소를 저장 (ARM)TTBR / (x86) CR3
(6) CPU 실행 => fetch => (ARM) PC / (x86) rip => 주소(VA) 가져오기 
    => 페이지폴트 처리 => 페이지(페이지 캐시) 할당 -> 디스크 I/O -> major fault
  **참고: MMU -> TLB 캐시 pte 확인 -> TTBR -> DRAM(페이지테이블 항목하나) 읽어온다 ** 
(7) MMU에게 가상주소를 넘긴다. 
(8) MMU 현재페이지 테이블을 찾아서 VA-PA 변환정보 읽어 PA반환 => 속도를 빠르게 하기 위해 캐싱(TLB)활용 
(8-1) 매핑된 PA가 없으면 Page Fault 발생, Page Fault를 일으킨 가상주소를 (ARM) FAR / (x86) CR2 레지스터에 입력
(9) 예외처리(Page Fault를 핸들링하기 위한) => entry => 예외처리 => Page Fault Handling
    - arch/x86/entry/entry_64.S: do_page_fault
    - VMA가 유효한가? => segfault or 
    - 페이지할당(페이지 테이블 요소 하나 PTE) 후 VA-PA 매핑(페이지테이블엔트리(한칸) 수정)
(10) 다시 MMU가 VA를 현재 변환테이블 기준으로 PA로 변환한다. 
(11) cache controller 통해서 CPU 내부 L1, L2, L3 접근하여 확인
(12) cache miss시 memory controller 통해서 DRAM 접근
    - int a를 자주 사용하는 경우 => prefetch() => L1에 미리 로드
(13) MOV 1 addr; => decode => execute => VA를 MMU에게 넘김 => 실패 => page fault
    스택공간 => 페이지 폴트처리 => 페이지(anonymoug page)할당 => VA-PA 정보 페이지테이블
    ** I/O 동반을 안하면 minor fault ** 
    
가상주소 전체를 user/kernel이 나눠씀 => user가 사용하는 가상주소 -> (VA-PA 필요할 때) 후매핑주소
                                     => kernel (주로) 사용하는 가상주소 -> (VA-PA) 선매핑 주소

**현재 리눅스 PageTable 레벨 확인하기** 

```bash
reallinux@ubuntu:~$ uname -a
Linux ubuntu 5.3.0 #2 SMP Fri Nov 1 04:44:53 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
reallinux@ubuntu:~$ cat /boot/config-5.3.0 | grep PGTABLE
CONFIG_PGTABLE_LEVELS=4
```

###### 실습 1. Pagefault 추적하기
```
sudo su
cd /sys/kernel/debug/tracing/
echo 1 > events/exceptions/page_fault_user/enable
cat trace_pipe
>bash-930   [003] d...  6288.896762: page_fault_user: address=0x55bc6c5557d0 ip=0x7ff74f7aa84c error_code=0x7
>bash-930   [003] d...  6288.896810: page_fault_user: address=0x55bc6c5506a0 ip=0x55bc6b805177 error_code=0x7

vim ${KERNEL_PATH}/arch/x86/include/asm/traps.h
> /Page fault error code
> 위의 error_code 비교하여 확인
```

**Tracing 종료하기** 
```
echo nop > current_tracer
echo 0 > options/stacktrace
echo 0 > events/enable
echo > trace
```

###### 실습2. Pagefault handler 주요흐름 추적

페이지 테이블은 kernel, anonymous, page cache로 분류
- anonymous page: stack, heap과 같이 파일과 연관되지 않은 페이지
- page cache: 디스크 블록을 유지해두는 용도 (파일과 연관됨)

page cache로 사용되는 경우가 anonymous 페이지로 사용되는 경우보다 많음. 
```
cd ${KERNEL_PATH}
vi arch/x86/entry/entry_64.S
> /trace_page_fault_entries
> /do_page_fault
> /__do_page_fault

Callstack은 아래와 같이 이루어짐

do_user_addr_fault()   : arch/x86/mm/fault.c
  -> handle_mm_fault()    :   mm/memory.c
      -> __handle_mm_fault()    :   mm/memory.c
          -> handle_pte_fault()     :   mm/memory.c
               -> ptep_set_access_flags()   :  arch/x86/mm/pgtable.c
                    -> set_pte() “페이지 테이블 한요소(pte) 수정”
```

#### 커널의 메모리 관리 구조
```plaintext    
+-----------+
|task_struct|
+-+---------+
  |
  | +---------+
  +>|mm_struct|
    ++--------+
     |
     | +------+  +------+  +-------+  +-----+
     +>|vma   +->|vma   +->|vma    +->|vma  |->
       |(heap)|  |(code)|  |(stack)|  |(...)|
       +------+  +------+  +-------+  +-----+
```

task_struct는 각 태스크(또는 스레드) 마다 만들어진다. 
mm_struct는 각 프로세스별로 만들어지며, 커널을 위한 init_mm이 별도로 존재함. 
커널을 위한 변환 페이지는 init_mm에 매핑되며 유저 프로세스에서 커널 영역에 fault발생 시 init_mm에서 조회를 시도함. 

페이지 테이블에 리스팅되어 있지만 물리주소가 없는 경우 => 페이지 폴트
페이지 테이블에 없는 가상주소를 로드 시도하는 경우 => 세그먼트 폴트
페이지 테이블 크기 4K == 4096 == 0x1000 = 12 bit == 2^12


###### 실습3. 내 프로그램 PageFault 추적

**cgdb 설치**
```bash
$ cd git
$ git clone https://github.com/cgdb/cgdb.git
$ cd cgdb
$ sudo apt-get install gdb texinfo libreadline-dev  
$ ./autogen.sh
$ ./configure
$ make -j4 && sudo make install
```

cgdb 사용하기 
- ESC: 커서 위로
- i: (gdb)커서 아래로
- :set dis  : 어셈블리어 보기
- :set nodies : 소스코드 보기

```bash
(gdb) b main
(gdb) run
(gdb) info proc
```

###### 커널 이벤트 추적하기

```bash
sudo su
cd /sys/kernel/debug/tracing/
echo 1 > events/exceptions/page_fault_user/enable
echo [PID] > set_event_pid
cat trace_pipe

cgdb 계속... 

(gdb) si
> 페이지 폴트가 발생할 때까지 반복
> 페이지 폴트가 발생한 주소를 찾아보기
(gdb) x/i 0x7ffff7a649f9
```

### Multi Level PageTable

32비트 리눅스 운영체제에서는 32비트만큼 가상 주소를 활용한다. 
총 주소는 pow(2, 32) 만큼 존재하고 페이지 테이블의 크기는 4KB(pow(2,12))이므로
페이지 테이블을 1단계로 변환하면 페이지 테이블이 pow(2,20)개 즉, 1M 개의 인덱스가 필요하다. 만약 운영체제가 64비트라면 64G개만큼의 인덱스가 필요하다. 이러한 비효율성을 극복하기 위하여 여러 단계로 매핑되는 다중 레벨 페이지 변환 구조를 리눅스에서 운용하고 있다. 

```plaintext
9Bit   9Bit   9Bit   9Bit        12Bit
+---+  +---+  +---+  +---------+  
|pgd|->|pud|->|pmd|->|pagetable|->Page
+---+  +---+  +---+  +---------+  
```

###### 실습 4. pagetable dump 실습

make menuconfig 통해서 pagetable dump 옵션 켜기

```
cd ~/git/linux
make menuconfig

#아래 커널 옵션 켜기
Kernel hacking --->
Export kernel pagetable layout to userspace via debugfs

make
sudo make install
sudo reboot

#재부팅 후
cd /sys/kernel/debug/page_tables/
cat current_user
cat current_kernel
cat kernel
```

###### 실습 5. /proc/self/maps 출력과정 추적

```bash
# cat 프로세스의 메모리 맵정보 (vma 여러개) 출력과정 추적하기
$ sudo uftrace record -d pid_maps.uftrace.data --force -K 30 /bin/cat /proc/self/maps 
$ cd pid_maps.uftrace.data
$ uftrace replay
```


**VSZ, RSS 확인**
```plaintext
root@ubuntu:/sys/kernel/debug/tracing# free -h
              total        used        free      shared  buff/cache   available
Mem:           1.9G        129M        1.0G        524K        848M        1.8G
Swap:          2.0G          0B        2.0G
root@ubuntu:/sys/kernel/debug/tracing# cat /proc/meminfo | head -5
MemTotal:        2035676 kB
MemFree:         1033364 kB
MemAvailable:    1868552 kB
Buffers:           40360 kB
Cached:           768748 kB
root@ubuntu:/sys/kernel/debug/tracing# man ps | grep RSS
       The SIZE and RSS fields don't count some parts of a process including the page tables, kernel stack, struct
       rss         RSS       resident set size, the non-swapped physical memory that a task has used (in
       rssize      RSS       see rss.  (alias rss, rsz).
root@ubuntu:/sys/kernel/debug/tracing# man ps | grep VSZ
       %z     vsz      VSZ
       vsize       VSZ       see vsz.  (alias vsz).
       vsz         VSZ       virtual memory size of the process in KiB (1024-byte units).  Device mappings are
root@ubuntu:/sys/kernel/debug/tracing#
root@ubuntu:/sys/kernel/debug/tracing# ps -eo pmem,rss,vsz,comm
%MEM   RSS    VSZ COMMAND
 0.4  8852  77676 systemd
 0.0     0      0 kthreadd
```

**Buff/Cache 비우기** 
```
root@ubuntu:/home/reallinux/git/linux/mm# cat /proc/meminfo | head -5
MemTotal:        2035676 kB
MemFree:         1296488 kB
MemAvailable:    1857320 kB
Buffers:           31868 kB
Cached:           529524 kB
root@ubuntu:/home/reallinux/git/linux/mm# echo 3 > /proc/sys/vm/drop_caches
root@ubuntu:/home/reallinux/git/linux/mm# cat /proc/meminfo | head -5
MemTotal:        2035676 kB
MemFree:         1761032 kB
MemAvailable:    1856944 kB
Buffers:            1332 kB
Cached:           106580 kB
. 현재 사용하고 있는(읽고 쓰고 있는) 파일 데이터는 비워지지 않음. 
```
###### 실습. mmap 시스템콜 추적실습 

*hello_mmap.c 파일 참조* 

```
reallinux@ubuntu:~$ vim mmap_test.txt
reallinux@ubuntu:~$ hexdump -x mmap_test.txt
0000000    3131    3131    3131    3131    3131    3131    3232    3232
0000010    3232    3232    3232    3232    3232    3332    3333    3333
0000020    3333    3333    3333    3434    3434    3434    3434    000a
000002f
reallinux@ubuntu:~$ ./hello_mmap mmap_test.txt
1111111111112222222222222223333333333344444444

reallinux@ubuntu:~$ hexdump -x mmap_test.txt
0000000    0000    0000    0000    0000    0000    0000    0000    0000
0000010    3232    3232    3232    3232    3232    3332    3333    3333
0000020    3333    3333    3333    3434    3434    3434    3434    000a
000002f
```

**mission: 아래 함수들을 찾아보자**

    1) arch/x86/mm/fault.c 의 do_page_fault() 함수 부터 시작
    2) arch/x86/mm/fault.c 의 do_user_addr_fault()
    3) arch/x86/mm/fault.c 의 find_vma()
    4) mm/memory.c 의 handle_mm_fault()
    5) mm/memory.c 의 __handle_mm_fault()
    6) mm/memory.c 의 handle_pte_fault()
    7) mm/memory.c 의 do_fault()
    8) mm/memory.c 의 do_read_fault()
    9) mm/memory.c 의 __do_fault()
    10) mm/memory.c 의 vma->vm_ops->fault(vmf);
    11) fs/ext4/inode.c 의 ext4_filemap_fault()

###### 실습 read 시스템콜 vs mmap 

- read 시스템콜은 파일을 파일을 열어 읽는 과정에서 일부 영역을 복사
- mmap 시스템콜은 파일을 열어 페이지 단위로 복사

*read.c 참조*
```
gcc -pg -g -o read read.c
$ sudo su
$ echo 3 > /proc/sys/vm/drop_caches
$ exit

$ sudo uftrace record -d read.uftrace.data -K 30 ./read
$ cd read.uftrace.data
# 문자열 검색으로 sys_read와 ext4_file_read_iter 찾기
$ uftrace replay
```
**mmap 실습 내용과 차이점을 찾아볼것**



### 유저 + 커널 가상주소 범위 알아보기

- 32Bit CPU의 가상주소 범위는 4GB(2^32)
    * User:Kernel 이 3:1 비율로 분할(3GB:1GB)하여 사용함(Configurable)
- 64Bit CPU의 가상주소 범위는 256TB(2^64)
    * User:Kernel 이 1:1 비율로 분할(128TB:128TB)하여 사용함
- 32Bit를 기준으로 보았을 때 기본값으로 커널은 1GB만큼의 가상주소를 할당받음.
- 만약 커널이 1GB이상의 주소를 핸들링해야하는 경우 커널의 가상주소 공간에서는 핸들링이 불가능함. 
- 이 경우 indirect하게 주소에 접근하여야 하며 이 경우를 별도로 High Zone으로 분류
- 커널의 메모리 존은 DMA, Normal, High로 분류
    * DMA: 외부 장치로부터 DMA를 통하여 값을 복사하는 영역
    * Normal: 기본 커널 영역
    * High: 커널이 direct하게 접근 불가능한 영역
    
### 커널 메모리
- 커널의 성능 향상을 위해 커널은 여러 메모리 영역을 가짐
  - DMA용 주소(16MB)
  - 선매핑용 주소(896MB)
  - 후매핑용 주소(128MB)
    - vmalloc
    - pkmap area
    - fixmap area
- 선매핑용 주소의 경우 페이지 폴트가 발생하지 않도록 미리 물리주소와 연결시킴
- 마스터 커널의 init_mm.pgd를 이용하여 커널 메모리를 관리
- 커널에서 메모리를 할당하는 방법은 아래와 같음
  - alloc_pages: 페이지 자체 할당
  - kmem_cache_alloc: 자주쓰는 자료구조
  - kmalloc: 물리연속보장 메모리
  - vmalloc: 물리연속보장하지 않는 메모리

###### 실습. alloc_page 호출 시 order 값 확인

```
$ sudo su
$ cd /sys/kernel/debug/tracing

# trace buffer 초기화
$ echo nop > current_tracer
$ echo 0 > events/enable
$ echo 0 > options/stacktrace
$ echo > trace

# alloc_page 함수 호출되는 시나리오 추적하기
$ echo  1 > events/kmem/mm_page_alloc/enable  
$ cat trace
...
bash-6967  [002] ....  5280.610411: mm_page_alloc: page=000000004bc90fd7 pfn=274176 order=3 ...  
```
###### 실습. alloc_page 호출 시 backtrace 정보 확인
```
$ sudo su
$ cd /sys/kernel/debug/tracing

# trace buffer 초기화
$ echo nop > current_tracer
$ echo 0 > events/enable
$ echo 0 > options/stacktrace
$ echo > trace

# alloc_page 함수 호출되는 시나리오 추적하기
$ echo  1 > events/kmem/mm_page_alloc/enable   
$ echo 1 > options/stacktrace
$ cat trace
```
- 케이스 1: alloc_pages_vma() 에서 alloc_page 하는 경우
- 케이스 2: pte_alloc_one() 에서 alloc_page 하는 경우
- 케이스 3: do_wp_page()에서 alloc_page

#### 버디 시스템

- 커널에서 메모리 파편화를 막고 최대한 메모리의 물리적 연속성 보장을 위해 페이지 테이블을 2의 승수 단위로 관리함. 
- 버디 정보 확인하기
```
cat /proc/buddyinfo
```
- 버디시스템은 order(2의 승수)별로 페이지 테이블의 큐를 가지고 있으며 요청하는 페이지 테이블 길이에 따라 페이지 테이블을 나누고 병합하여 요구하는 사이즈 길이 만큼의 연속됙 페이지 테이블을 반환
- 버디 시스템에서 페이지 테이블을 받아내는 함수가 rmqueue임
- **show_buddyinfo.py** 참조
- ~/git/linux/mm에서 rmqueue함수를 찾아보자

#### 슬랩할당자

- 페이지 테이블 내부에서의 데이터 파편화 및 사용 효율성을 높이기 위해 사용되는 방법
- 페이지 내부에 자주 사용되는 구조체 풀을 미리 만들어 놓아 요청 시 할당
- 커널에서 자주사용되는 구조체에 대하여 slab을 활용
- slab을 운용하는 오버헤드를 줄이기 위하여 slob, slub 등을 사용함
- kmem_cache_create 함수를 이용하여 객체창고 초기화
- kmem_cache_alloc 함수를 이용하여 객체 할당
- kmem_cache_free 함수를 이용하여 객체 반환
```bash
$cat /proc/slabinfo
$cd ~/git/linux
$vim include/linux/slab_def.h
$vim include/linux/slub_def.h
```

**buddy와 slab의 차이? --> buddy는 페이지 단위에서의 파편화 개선, slab는 페이지 내부에서의 파편화 개선**

#### vmalloc
- vmalloc은 연속된 가상주소를 물리적으로 연속되어있지 않은 메모리에 할당하는 함수임.

```
$ cd ~/git/linux/mm
# vmalloc 의 주요기능함수 __vmalloc_node_range() 에서
# 주요 역할 함수 두가지
# (1) __get_vm_area_node() 을 통한 가상주소관리 자료구조 vm_struct 할당
# (2) __vmalloc_area_node() 을 통한 페이지할당(alloc_page) 및 커널페이지테이블에 매핑작업
$ vim vmalloc.c
...
void *__vmalloc_node_range(unsigned long size, unsigned long align,
                        unsigned long start, unsigned long end, gfp_t gfp_mask,     
                        pgprot_t prot, unsigned long vm_flags, int node,
                        const void *caller)
{
        struct vm_struct *area;
...

        area = __get_vm_area_node(real_size, align, VM_ALLOC | VM_UNINITIALIZED |
                                vm_flags, start, end, node, gfp_mask, caller);
...

        addr = __vmalloc_area_node(area, gfp_mask, prot, node);
...
```


```
$ cd ~/git/linux/mm
# __vmalloc_area_node() 함수 안에서 물리페이지프레임 + 커널페이지테이블 매핑작업
# 주요 역할 함수 두가지
# (1) alloc_page 또는 alloc_page_node() 를 통한 물리 페이지 프레임 할당
# (2) map_vm_area() 을 통한 커널 페이지 테이블 수정작업 (가상주소와 물리페이지프레임 매핑)
$ vim vmalloc.c
...
static void *__vmalloc_area_node(struct vm_struct *area, gfp_t gfp_mask,
                                 pgprot_t prot, int node)
{
        nr_pages = get_vm_area_size(area) >> PAGE_SHIFT;
...
        area->pages = pages;
        area->nr_pages = nr_pages;

        for (i = 0; i < area->nr_pages; i++) {
                struct page *page;

                if (node == NUMA_NO_NODE)
                        page = alloc_page(alloc_mask|highmem_mask);
                else
                        page = alloc_pages_node(node, alloc_mask|highmem_mask, 0); 
...
        if (map_vm_area(area, prot, pages))
```

**vmalloc 호출 시 동작**

1. addr = vmalloc(8192); 호출
2. 사용하려는 vmalloc용 가상주소 준비
3. 사용하려는 페이지 할당 alloc_pages()
4. VA-PA 매핑(연결) -> 
    (현재 페이지 테이블이 아닌 마스터 페이지 테이블)페이지 테이블에 set_pte()
    * addr을 사용하려고 했을 때 page fault가 발생할 수 있다. *
    - 현재 페이지 테이블 영역에는 VA-PA정보가 없기 때문
5. addr 사용 시 page fault 가 발생
6. 페이지 폴트 처리 => 커널 주소 => vmalloc 주소 처리 => vmalloc_fault 처리
7. 마스터커널 페이지테이블에서 init_mm 복사 => 현재 페이지 테이블에 적는다.

###### 실습. vmalloc의 stacktrace 추적하기

```bash
$ sudo su
$ cd /sys/kernel/debug/tracing

# trace 초기화
$ echo 0 > events/enable
$ echo > trace

$ echo nop > current_tracer
$ echo 'p:vmalloc vmalloc' > kprobe_events
$ echo 1 > events/kprobes/vmalloc/enable
$ echo 1 > options/stacktrace

$ cat trace_pipe

# 다른 터미널에서 BPF 테스트
$ cd ~/git/linux/samples/bpf/
$ sudo ./tracex3
```

*vmalloc callstack*
```plaintext
vmalloc
 -> __vmalloc_node_flags
  -> __vmalloc_node
   -> __vmalloc_node_range
    -> __get_vm_area_node // 가상주소 관리를 위한 vm_struct 할당
     -> __vmalloc_area_node // 필요한 size 만큼 page 할당해서 vm_struct에 기록
      -> map_vm_area // 시작addr 부터  끝 addr 까지 매핑(pagetable setting)
       -> vmap_page_range
        -> vmap_page_range_noflush // 마스터 커널 페이지 테이블에 pte 세팅
```

# 3일차

### Swap 영역

메모리가 적게 남았을 때 메모리를 확보하기 위하여 kswapd 함수를 사용하여 유휴 페이지를 스왑영역에 기록함

kswapd 함수가 동작하는 조건

- Low단계에서 시작
- Min일 때 direct reclaim
- High일때 종료

**메모리 Zone 살펴보기(DMA32)*

- min  워터마크 min (direct reclaim 을 허용하는경우는 회수후 할당 가능)
- low: 워터마크 low (현재 가용가능한 범위가 이 이하로 내려가면 kswapd wakeup)
- high: 워터마크 high (kswapd sleep 되는 지점)
- spanned = Zone의 전체 페이지 수: zone_end_pfn - zone_start_pfn;
- present = 실제 존재하는 페이지 수: spanned_pages - absent_pages(pages in holes);
- managed = 예약된 페이지 제외한 버디시스템으로 관리가능한 페이지수 present_pages - reserved_pages;

```
# 64bit 기준 사실상 Normal 처럼 활용 되고 있는 DMA32 기준으로 
# Zone 정보 살펴보기
$ cat /proc/zoneinfo | grep -A 7 DMA32  
Node 0, zone    DMA32
  pages free     271567
        min      1402
        low      1898
        high     2394
        spanned  520176
        present  520176
        managed  501076
```

###### 실습: 페이지 회수과정 tracing(kswapd 추적)

**kswapd wake/sleep 추적**

```bash
# tracing 준비 및 초기화
$ sudo su
$ cd /sys/kernel/debug/tracing
$ echo 0 > events/enable
$ echo 0 > options/stacktrace
$ echo > trace

# kswapd 프로세스 wake / sleep 추적하기
$ echo 1 > events/vmscan/mm_vmscan_kswapd_sleep/enable 
$ echo 1 > events/vmscan/mm_vmscan_kswapd_wake/enable 
$ echo 1 > events/vmscan/mm_vmscan_wakeup_kswapd/enable  
$ cat trace_pipe

# 파이썬 numpy 라이브러리 설치
$ sudo apt-get install python-pip
$ sudo pip install numpy

# swap 유도하는 파이썬 스크립트 실행하기
$ vim swap_test.py
$ cat swap_test.py

#!/usr/bin/python2
import numpy

result = [numpy.random.bytes(1024*1024*2) for x in xrange(1024)]  
print len(result)
 
 # swap 유도 테스트 하기
 $ python swap_test.py
```

**페이지 회수과정 event 추적**
```bash
$ sudo su
$ cd /sys/kernel/debug/tracing
$ echo 0 > events/enable
$ echo 0 > options/stacktrace
$ echo > trace

* 페이지 회수(reclaim) 과정 추적하기 pageout -> writepage
$ echo 1 > events/vmscan/mm_vmscan_direct_reclaim_begin/enable  
$ echo 1 > events/vmscan/mm_vmscan_writepage/enable 
$ echo 1 > events/vmscan/mm_vmscan_direct_reclaim_end/enable 
$ echo 1 > events/oom/enable

$ cat trace_pipe

* 또 다른 터미널에서 swap 유도 테스트
$ python swap_test.py
```

###### 페이지 회수과정 Callstack

```plaintext
kswapd
 -> balance_pgdat
   -> kswapd_shrink_node
     -> shrink_node
       -> shrink_node_memcg
         -> shrink_list
           -> shrink_inactive_list
             -> shrink_page_list
               -> pageout
```

```plaintext
__alloc_pages
 -> __alloc_pages_nodemask
  -> __alloc_pages_slowpath
   -> __alloc_pages_direct_reclaim
    -> __perform_reclaim
     -> try_to_free_pages
      -> do_try_to_free_pages
       -> shrink_zones
        -> shrink_node
         -> shrink_node_memcg
          -> shrink_list
	   -> shrink_inactive_list
	    -> shrink_page_list
	     -> pageout
```

### ARMv8 Architecture

- x0 "함수 리턴값를 담는 용도로 쓰이는 레지스터"
- x0~x7 "함수 인자를 담는 용도로 쓰이는 레지스터"
- x8 "시스템콜 넘버 저장용 / 리턴값 클때 해당 메모리 위치 주소값 적는 용도"
- x16~x17 "인트라 프로시저 레지스터: 라이브러리 함수 호출시 활용"
- x29 (Frame Pointer) "함수 단위로 스택의 시작주소: 함수단위 땅 시작주소"
- x30 (Link Register) "함수가 끝나고 돌아갈주소를 저장: Return Adress"
- w0 ~ w30 "x0 ~ x30와 동일한 레지스터로 32bit 크기 레지스터로 나눠쓸때 사용"
- v0~v31 "FP / SIMD 전용 레지스터"
    - Bn : Byte(8bit), 
    - Hn : Half(16bit), 
    - Sn : Single(32bit), 
    - Dn : Double(64bit), 
    - Qn : (128bit)
- CLIDR(Cache Level ID Register): 캐시 레벨 설정 레지스터
- CTR(Cache Type Register): 캐시라인 크기 판결 레지스터
- TCR(Translation Control Register): VA-PA 변환과정 컨트롤 레지스터
- TLB(Translation Lookaside Buffer): VA-PA 변환정보 캐시
- TTBR0 : 유저 전용 변환테이블 베이스 주소 레지스터
- TTBR1 : 커널 전용 변환테이블 베이스 주소 레지스터
- 메모리 변환시 가상 주소를 물리주소로 변환하는 과정이 모두 끝난 뒤에 캐시 시스템에 접근하여 캐싱되었는지의 여부를 확인



## File System & VFS

**High Level Architecture**

```plaintext
+----------------------------------+
|User                              |
|Space                             |
|   +----------------------------+ |
|   |     User Application       | |
|   +------+-------------+-------+ |
|          |             |         |
|          |      +------+-------+ |
|          |      |GNU C Library | |
|          |      +------+-------+ |
|          |             |         |
+----------+-------------+---------+
|Kernel    |             |         |
|Space     |             |         |
|   +------+-------------+-------+ |
|   |    System call Interface   | |
|   +-------------+--------------+ |
|                 |                |
|   +-------------+--------------+ |
|   |    Virtual File System     | |
|   +------+-----------------+---+ |
|          |                 |     |
|   +------+-----------------+-+   |
|   |    Inode/Dentry cache    |   |
|   +------+-----------------+-+   |
|          |                 |     |
|   +------+---------------+ |     |
|   |Individual File System| |     |
|   +------+---------------+ |     |
|          |                 |     |
|   +------+-----------------+-+   |
|   |     Buffer/Page Cache    |   |
|   +------+-----------------+-+   |
|          |                 |     |
|   +------+-----------------+--+  |
|   |    Generic Block Layer    |  |
|   +-------------+-------------+  |
|                 |                |
|   +-------------+-------------+  |
|   |      Device Driver        |  |
|   +---------------------------+  |
|                                  |
+----------------------------------+
```

- VFS: Virtual File System, Inode/Dentry cache
- FS: Individual File System, Buffer/Page cache
- Block: Generic Block Layer, Device Driver


- Buffers: 보조 기억장치의 내용을 주기억 장치에 복사
- Cached: Buffers를 관리하기위한 메타데이터를 주기억장치에 저장 

#### write()
- Buffered/IO
  - write_begin: pagecache 준비
  - 유저 데이터 복사: pagecache 쓰기
  - write_end: pagecache 더티 flag set
- write 과정 : User -> Syscall -> VFS -> ext4(FS) -(비동기)-> Device
- FS 함수에서 Buffered I/O 에 기록한 뒤 dirty비트를 세트하고 종료
- writeback함수에서 추후 device에 전송(약 5초 주기로 실행)
- 파일 정보는 Buffer, 파일 내용은 Cache로 분류됨

**파일 관리**
- 파일 open 시
- 유저 태스크에 대한 커널의 task_struct내부에 fd table 존재
- fd 테이블에 struct file이 매핑
- struct file 내부에 struct inode가 있으며 inode struct address_space를 포함
- address_space내부에 page들 포함

```bash
# fclose 밑에서 시스템콜 write가 호출 되었음을 확인하자
$ cd write.uftrace.data
$ uftrace replay -F fclose -t 20us
# DURATION     TID     FUNCTION
            [  4081] | fclose() {
            [  4081] |   __x64_sys_write() {
            [  4081] |     ksys_write() {
            [  4081] |       vfs_write() {
            [  4081] |         __vfs_write() {
            [  4081] |           new_sync_write() {
            [  4081] |             ext4_file_write_iter() {
            [  4081] |               __generic_file_write_iter() {
            [  4081] |                 generic_perform_write() {
            [  4081] |                   ext4_da_write_begin() {
            [  4081] |                     __block_write_begin() {
  22.512 us [  4081] |                       __block_write_begin_int(); 
  22.840 us [  4081] |                     } /* __block_write_begin */
  42.582 us [  4081] |                   } /* ext4_da_write_begin */
            [  4081] |                   ext4_da_write_end() {
  24.532 us [  4081] |                     generic_write_end();
  ...
```

#### read()
1. 페이지 캐시 탐색(pagecache_get_page) -> hit -> read
2. miss인 경우
    1. 페이지 캐시 준비
    2. read_pages() -> I/O 요청
    3. 블록 layer 처리
    4. 블록 디바이스 드라이버 SCSI
    5. (readahead)
    6. schedule(); 호출 "CPU를 놓는다"
    7. DISK IRQ 발생 => wakeup

- read 과정: User 함수 → Syscall 함수→ VFS 함수 → ext4 함수 흐름
```bash
# pagecache hit 케이스
# ext4_file_read_iter 이후에 touch_atime()을 통해서
# 파일 access time 변경하는 함수 확인하기
$ cd read.uftrace.data
$ uftrace replay -F fread -t 30us
# DURATION     TID     FUNCTION
            [  4085] | fread() {
            [  4085] |   __x64_sys_read() {
            [  4085] |     ksys_read() {
            [  4085] |       vfs_read() {
            [  4085] |         __vfs_read() {
            [  4085] |           new_sync_read() {
            [  4085] |             ext4_file_read_iter() {
            [  4085] |               generic_file_read_iter() { 
  33.108 us [  4085] |                 touch_atime();
 ...
```
- read2 과정: User 함수 → Syscall 함수→ VFS 함수 → ext4 함수 → Block 함수흐름 
```bash
# pagecache miss 케이스: ext4_file_read_iter 이후에 read_pages()을 통해서
# ext4_readpages()를 호출하고 Block I/O 요청을 하는 흐름까지 이어지는것을 확인하자
$ cd read2.uftrace.data
$ uftrace replay -F fread -t 31us
# DURATION     TID     FUNCTION
            [  4101] | fread() {
            [  4101] |   __x64_sys_read() {
            [  4101] |     ksys_read() {
            [  4101] |       vfs_read() {
            [  4101] |         __vfs_read() {
            [  4101] |           new_sync_read() {
            [  4101] |             ext4_file_read_iter() {
            [  4101] |               generic_file_read_iter() {
...
            [  4101] |                       read_pages() {
            [  4101] |                         ext4_readpages() {
  31.141 us [  4101] |                           ext4_mpage_readpages();
  31.472 us [  4101] |                         } /* ext4_readpages */
...
            [  4101] |                                 blk_mq_run_hw_queue() {
  31.420 us [  4101] |                                   __blk_mq_delay_run_hw_queue();
  32.230 us [  4101] |                                 } /* blk_mq_run_hw_queue */
...
            [  4101] |                 io_schedule() {
            [  4101] |                   schedule() {
...
```
- 함수 흐름

```plaintext
read() -> __vfs_read() --> 
ext4_file_read_iter() <--> pagecache_get_page()
--NO PAGE--> page_cache_sync_readahead() --> read_pages()
--> ext4_readpages() [Store Callback function(bi_endio)]
--> summit_bio() --> scsi_dispatch_cmd() --> DISK OPERATION

**IRQ CALLED** --> __do_softirq() --> bio_endio()
```

- 분기 과정(ext4, proc, ramdisk 등)
```
# VFS -> FS (ext4) 처리 흐름 확인 하자
$ cd write.uftrace.data
$ uftrace replay -F fclose -t 10us
# DURATION     TID     FUNCTION
            [  4081] | fclose() {
            [  4081] |   __x64_sys_write() {
            [  4081] |     ksys_write() {
            [  4081] |       vfs_write() {
            [  4081] |         __vfs_write() {
            [  4081] |           new_sync_write() {
            [  4081] |             ext4_file_write_iter()   {
...
```
```
# VFS -> shmem 처리 흐름 확인 하자
$ cd ramdisk_write.uftrace.data
$ uftrace replay -F ksys_write@kernel -t 1ms
# DURATION     TID     FUNCTION
            [  1564] | ksys_write() {
            [  1564] |   vfs_write() {
            [  1564] |     __vfs_write() {
            [  1564] |       new_sync_write() {
            [  1564] |         generic_file_write_iter() {
            [  1564] |           __generic_file_write_iter() {
            [  1564] |             generic_perform_write() {
            [  1564] |               shmem_write_begin() {
            [  1564] |                 shmem_getpage_gfp.isra.57() {  
            [  1564] |                   shmem_alloc_page() {
...
```
```
# VFS -> proc 처리 흐름 확인 하자
$ cd devices.uftrace.data
$ uftrace replay -F read -t 100ms
# DURATION     TID     FUNCTION
            [  3211] | read() {
            [  3211] |   __x64_sys_read() {
            [  3211] |     ksys_read() {
            [  3211] |       vfs_read() {
            [  3211] |         __vfs_read() {
            [  3211] |           proc_reg_read() {
            [  3211] |             seq_read() {
            [  3211] |               devinfo_show() {  
...
```

### schedule()

- 능동적 호출 : 일부러 CPU 점유권을 놓는다. 
- 수동적 호출(되는 경우): 현재 프로세스는 원하지 않는데 CPU 점유권을 뺏기는 경우 => 선점

#### open()

- task_struct 별로 관리되는 파일 리스트인 fd table 에 파일 등록
- do_sys_open 시스템콜의 기능
  - 커널 file 객체 준비
    - namei 작업(경로명을 inode로 변환)
    - 디렉토리 내 검색
    - 파일 inode 생성
    - file객체에 inode연결
  - File Descriptor 제공
    - File Descriptor 준비
    - fdtable[fd]에 file객체 저장
    - fd 인덱스 값 리턴
- dentry 객체: 경로를 효율정으로 다루기 위한 커널 데이터
  - e.g. / -> usr -> local -> bin -> uftrace
- open 가능한 최대 파일 개수
  - $ cat /proc/sys/fs/file-max
  - $ ulimit -a | grep files
- \>과 |를 이용한 리디렉션
  - dup2함수를 이용하여 fd를 바꾸는 동작을 내부적으로 수행

#### sync()

- 함수 호출 과정

```plaintext
sync() --> ksync() 
---> wakeup_flusher_threads() -> wb_start_writeback()
 |                              -> ext4_writepages()
 +-> flush metadata --> iterate_supers()        |
                       -> iterate_bdevs()       |
                                |               |
                                v               |
                        blkdev_writepages()     |     
                                |               |
                                v               |
                            sumit_bio() <-------+
```

###### 실습. writeback 함수 호출과정 추적
```bash
$ sudo su
$ cd /sys/kernel/debug/tracing

# timer interrupt 함수 추적 끄기
$ echo smp_apic_timer_interrupt > set_graph_notrace

# writeback 함수 필터
$ echo wb_workfn > set_graph_function
$ echo function_graph > current_tracer

$ cat trace
```

```
아래 순서로 함수를 소스코드에서 살펴본다.
  1. wb_workfn
  2. wb_do_writeback
  3. wb_writeback (trace_writeback_exec)
  4. __writeback_inodes_wb
  5. writeback_sb_inodes
  6. __writeback_single_inode (trace_writeback_single_inode_start)
  7. do_writepages
```

#### File System

- 파일 시스템이란?
    - 물리적인 디스크 블록들을 어떻게 read/write 할 지에 대한 전략
- 파일 시스템 포맷을 하면 데이터 블록이 전부 날라가나요? 
    - 포맷 시 슈퍼블록만 초기화하므로 복구가능
- 물리적인 슈퍼블록과 커널에서 관리를 위한 struct super_block을 구분해야
    - struct ext4_super_block <-> struct super_block
- 또한 물리적인 inode와 커널에서 관리를 위한 struct inode 또한 구분하여야
    - struct ext4_inode <-> struct inode
    
    
##### Block Layer

**주요 자료구조**

- struct bio
- struct request
- struct request_queue
- 자료구조 구성: request_queue -1...n-> request -1...n-> bio
- request를 줄이기 위해서 연속된 영역의 bio에 대해서는 merge를 수행.
    * 기존 request에 비교하여 새 request가 앞 영역을 요청하는 경우 front_merge 
    * 기존 request에 비교하여 새 request가 뒷 영역을 요청하는 경우 back_merge
    * 순차로 요청하는 경우 back_merge가 일어나므로 front_merge보다 back_merge가 많이 일어남

** Block 처리 Trace Action 정리 ** 

- A: [Remap] 파티션 간(e.g. sda2 -> sda) 리매핑
- Q: [Queue] Block I/O 처리 단위 struct bio enq 시점
- G: [Getrequest] bio를 담는 request 생성
- I: [Insert] Block I/O SW큐에 request추가
- D: [Dispatch] HW큐의 request들을 Block device driver로
- C: [Complete] 디스크의 처리가 끝난 뒤 인터럽트 후처리시첨 upcall
- P: [Plug]
- U: [Unplug]
- M: [Merge]

** Block 처리 Traction Action 예시 **

- A -> Q -> M : 기존 request가 있으며 병합 가능하여 병합 후 종료
- A -> Q -> G -> I -> D -> C

**Block Layer summit_bio 이후 처리과정** 
```plaintext
                          +-------------+
                          v    [A]      |
+------+               +-----+   +------+-----+
| File |  struct bio   | bio |   |generic_make|
|System+-------------->|     +-->| request[A] |
+------+               +-----+   +-+----------+
   ^                               |
   |[C]                    +-------+[Q]
   |bio_endio              v
+--+---+             +------------+   +-------+
|block |             |make_request|[G]|request|
|device|             |     _fn    +-->|       |
|driver|             +-----+---+--+   +-----+-+
+------+                [M]|   |  [M]       |[P]
   ^                       |   +--------+   |
   |                       v            v   v
   |     +--------+   +---------+     +-------+
   | [D] |dispatch|   |   I/O   |[I|U]|plugged|
   +-----+  queue |<--+Schedular|<----+  list |
         +--------+   +---------+     +-------+
```



###### 파일의 블록처리과정 추적하기
```
reallinux@ubuntu:~/test$ ./write
reallinux@ubuntu:~/test$ ls -i hello.txt
955785 hello.txt
--------------
reallinux@ubuntu:~$ sudo btrace /dev/sda > write.btrace.data
^C
reallinux@ubuntu:~$ sudo istat /dev/sda2 955785
inode: 955785
Allocated
Group: 116
Generation Id: 3674162498
uid / gid: 1000 / 1000
mode: rrw-rw-r--
Flags: Extents,
size: 23
num of links: 1

Inode Times:
Accessed:       2022-06-16 00:14:19.438377678 (UTC)
File Modified:  2022-06-16 00:14:19.438377678 (UTC)
Inode Modified: 2022-06-16 00:14:19.438377678 (UTC)
File Created:   2022-06-16 00:14:19.438377678 (UTC)

Direct Blocks:
3707133

Note: 29657064 = 3707133 * 8
reallinux@ubuntu:~$ cat write.btrace.data | grep 29657064
  8,0    1      129     9.927317017  1128  A  WS 29661160 + 8 <- (8,2) 29657064 <= /dev/sda2 to /dev/sda
reallinux@ubuntu:~$ ^C
reallinux@ubuntu:~$ cat write.btrace.data | grep 29661160
  8,0    1      129     9.927317017  1128  A  WS 29661160 + 8 <- (8,2) 29657064
  8,0    1      130     9.927318057  1128  Q  WS 29661160 + 8 [kworker/u8:0]
  8,0    1      131     9.927320279  1128  G  WS 29661160 + 8 [kworker/u8:0]
  8,0    1      134     9.927325061  1128  I  WS 29661160 + 8 [kworker/u8:0]
  8,0    1      136     9.927348379  1128  D  WS 29661160 + 8 [kworker/u8:0]
  8,0    2       63     9.927478548     0  C  WS 29661160 + 8 [0]
reallinux@ubuntu:~$
```


*kworker를 사용하는 이유*
- 후반부 작업 처리 로직
- Entry(인터럽트, 예외처리) 우선순위가 높게 처리되는 작업
- 전반부 작업 => 필수적인 것만 먼저 처리하고
- 나중에 해도 되는 것은 => 후반부 작업 처리로 미룬다
- 후반부 작업 처리에도 시급히 처리해야하는 것과 천천히 처리해야하는 것들이 있으므로
- kworker의 nice값이 다르다. 

## Schedular

### 스케쥴러의 기본 동작 방식 이해

- scheduler 태스크 분배이해
  - 스케줄러는 starvation을 막기 위해 여러 태스크에 시간을 분배
  - 이를 위해 타임슬라이스(배정한 시간)과 런타임(실행한 시간)을 관리
  - 이를 period의 비율로 관리하는데 태스크 숫자가 많아지면 너무 자주 context switching이 발생
  - 이를 막기위해 최소 실행단위인 sched_min_granularity가 있음.

```bash
# nansecond 단위 (ns)
# 2,250,000 ns == 2.25 ms
$ cat /proc/sys/kernel/sched_min_granularity_ns   
2250000

# 분배할 CPU 총시간 period 기본값 100 ms 이기때문에
# 총 100 ms 중에서 한 프로세스당 최소 2.25 ms 는 보장해준다  
# (프로세스 개수가 무한이 많아져도 보장)
$ cat /sys/fs/cgroup/cpu/cpu.cfs_period_us 
100000
```

  - 태스크에 따라 중요한 태스크에 더 많은 시간을 분배하고자 함
  - nice(호구력)에 따라 최소 -20 부터 19까지 값을 분배
  - nice값이 낮을수록 높은 우선순위, 높을수록 낮은 우선순위
    - nice 값에 따라 timeslice로 실행시간을 부여하는 경우
    - 0:100ms .... 19:5ms로 매핑을하면
    - 0과 1은 100ms:95ms로 거의 공평하나
    - 19와 20은 10ms:5ms로 매우 불공평
    - 이를 위해 가중치 기반으로 timeslice를 부여
    - timeslice = period * (current_weight)/(sum_weight)
  - 스케쥴링 시 여러 태스크 중 어떤 태스크를 실행할 것인가?? 
    - Option1: 실행한 시간이 적은것
      - Pro: 최소 실행시간 보장
      - Con: 우선권 침해, 연산 복잡
    - Option2: 우선순위 높은 태스크
      - Pro: 우선권 보장
      - Con: Starvation
    - BestPick? --> 우선순위에 따라 실행한 시간을 조정(virtual runtime)
      - vruntime = runtime * nice_0_weight/my_weight

```bash
$ cd ~/git/linux/
$ vim kernel/sched/core.c
7526 const int sched_prio_to_weight[40] = {
7527  /* -20 */     88761,     71755,     56483,     46273,     36291,
7528  /* -15 */     29154,     23254,     18705,     14949,     11916,
7529  /* -10 */      9548,      7620,      6100,      4904,      3906,
7530  /*  -5 */      3121,      2501,      1991,      1586,      1277,
7531  /*   0 */      1024,       820,       655,       526,       423,
7532  /*   5 */       335,       272,       215,       172,       137,
7533  /*  10 */       110,        87,        70,        56,        45,
7534  /*  15 */        36,        29,        23,        18,        15,   
7535 };
```

### 스케줄러 구성 요소 4가지

1. schedular_tick: CPU timer IRQ 마다 타임슬라이스 체크 및 LB시도
   1. update_curr(): 실행한 시간 계산
   2. check_preempt_tick(): 선점여부 확인
   3. trigger_load_balance(): CPU 사이 로드 밸런싱
2. schedule: next 프로세스 선택 및 context switch
   1. pick_next_task(): 최소 runtime 태스크 선택
   2. context_switch(): 다음 task가 CPU 점유
3. try_to_wake_up(ttwu): sleep 중인 특정 프로세스 wake up
   1. activate_task(): 런큐에 넣기
   2. ttwu_do_wakeup(): 태스크 Ready 상태로 변경
4. load_balance: Timer irq 마다 cpu들 사이 runqueue 로드밸런싱
   1. find_busiest_queue(): 가장 바쁜 런큐 찾기
   2. detach_tasks(): 태스크 빼내기
   3. attach-tasks(): 태스크 넣기

인터럽트/ 예외처리 복귀직전
에 need_resched 체크하는 코드
schedule(); // 강제적으로 수동적으로 불려진 것
syscall read() => I/O 요청 => schedule();
IRQ 핸들러함수 => ttwu 함수 구현

current(): 현재 프로세스 컨텍스트를 가져오기
ret_from_fork(): main함수 이전에 불려지는 최초 함수. 

context_switch() => CPU에 프로세스를 탑승(로드)시킨다 
context_switch() => prev -> next (유저/커널 태스크 간 컨텍스트 스위치)

1. 한번도 실행한적없고 최초로 실행하는 경우 => 무조건 ret_from_fork()부터 시작

A => B 태스크
- 본인이 스스로 CPU 점유를 놓은 경우(예: read() I/O 결과를 받기 위해 schedule())
- 본인은 원하지 않았는데 강제로 CPU를 뺏기는 경우(예: mysql 구동 -> IRQ -> 복귀직전)

2. 전에 한번 실행했었고 다시 실행하는 경우

B => C => A => B
switch_to(prev, next, prev) => CPU에 B 태스크를 다시 탑승시켜서 (복귀)

- ( B => C 에서)
    prev = B
    next = C

- (A => B 에서)
    prev = A
    next = C

B입장에서 돌아온 경우 이전 컨텍스트를 복귀하였으므로 prev가 자신이 되어있음. 


rdx(A) => rax(A) => rdi(A)



#### runqueue enq

- runqueue에 언제 task가 들어갈까?(enqueue)
1. fork() 수행 시
2. try_to_wake_up()을 통해서 runqueue에 task가 들어갈 수 있다. 

- runqueue에서 언제 task가 빠져나갈까(dequeue)
1. 이벤트를 기다릴때


sshd(서버) <- ssh(클라이언트)

1. 서버가 클라이언트의 패킷을 기다린다(sleep)
2. 패킷 도착! => DRAM => DMA 영역에 복사
3. 인터럽트 발생!! => do_IRQ();
4. 네트워크 패킷 인터럽트 처리
    (네트워크의 경우 전반부 0.1% / 후반부 99.9%)
5. 후반부 작업에서 실제 패킷을 parsing하는 작업을 수행한다. 
    ethernet header
    IP header
    TCP header
6. 데이터 준비 완료 => tcp_data_ready()
7. sock_def_readable()
8. ttwu try_to_wake_up


```
=> enqueue_task_fair
=> ttwu_do_activate
=> try_to_wake_up >> runqueue에 넣음
=> default_wake_function
=> pollwake
=> __wake_up_common
=> __wake_up_common_lock
=> __wake_up_sync_key
=> sock_def_readable
=> tcp_data_ready
=> tcp_data_queue
=> tcp_rcv_established
=> tcp_v4_do_rcv
=> tcp_v4_rcv
=> ip_protocol_deliver_rcu
=> ip_local_deliver_finish
=> ip_local_deliver
=> ip_sublist_rcv_finish
=> ip_sublist_rcv
=> ip_list_rcv
=> __netif_receive_skb_list_core
=> netif_receive_skb_list_internal
=> gro_normal_list.part.132
=> napi_complete_done
=> e1000_clean
=> net_rx_action
=> __do_softirq
=> irq_exit
=> do_IRQ
=> ret_from_intr
=> native_safe_halt
=> default_idle
=> arch_cpu_idle
=> default_idle_call
=> do_idle
=> cpu_startup_entry
=> start_secondary
=> secondary_startup_64
=> 0
=> 0
=> 0
=> 0
=> 0
=> 0
=> 0
=> 0
           sshd-1417    [002] d... 16634.918520: enqueue_task_fair: (enqueue_task_fair+0x0/0x590)
           sshd-1417    [002] d... 16634.918525: <stack trace>
```


## Signal

1. 시그널 전송 enq(generate)
   - 시스템콜 전달시 task_struct의 sigpending에 enqueue
   - struct sigqueue 사용
2. 시그널 처리 deq(deliver)
   - task_struct에서 sighand_struct에
   - task에서 사용할 signal핸들러가 등록되어 있음
   - 없는 경우 종료가 default로 설정됨

- 시그널 처리 자체는 유저 태스크에서 하지만 시그널 전달과 함수는 커널에 등록되어 있음. 
- 커널에서 시그널 함수를 직접 부를 수 없음 --> 어떻게 시그널 함수를 유저 태스크에서 실행할까? 
  - sighand_struct에서 유저 태스크의 handle_sigint()함수를 실행
  - handle_sigint()는 시스템콜 sigreturn()을 실행
  - 이후 등록한 signal_test 함수 실행
  - 
### 시그널 전달

kill -> 시그널(9) -> mysql
mysql -> task_struct -> signal pending queue[] <= 9
mysql -> IRQ -> IRQ 복귀코드 -> retint_user -> 선점여부 체크/시그널 체크


###### 실습. top 종료

```bash
root@ubuntu:/home/reallinux/kill_sig.uftrace.data# cd /sys/kernel/debug/tracing/
root@ubuntu:/sys/kernel/debug/tracing# echo nop > current_tracer
root@ubuntu:/sys/kernel/debug/tracing# echo 0 > options/stacktrace
root@ubuntu:/sys/kernel/debug/tracing# echo 0 > events/enable
root@ubuntu:/sys/kernel/debug/tracing# echo > trace
root@ubuntu:/sys/kernel/debug/tracing# echo 1 > events/signal/enable
root@ubuntu:/sys/kernel/debug/tracing# echo 1 > options/stacktrace
root@ubuntu:/sys/kernel/debug/tracing# pidof top
root@ubuntu:/sys/kernel/debug/tracing# pidof top
1878
root@ubuntu:/sys/kernel/debug/tracing# echo 1878 > set_event_pid
root@ubuntu:/sys/kernel/debug/tracing# echo > trace
root@ubuntu:/sys/kernel/debug/tracing# cat trace_pipe
             top-1878    [003] d...  1094.298909: signal_deliver: sig=20 errno=0 code=128 sa_handler=560399ad65e0 sa_flags=4000000
             top-1878    [003] d...  1094.298919: <stack trace>
 => get_signal
 => do_signal
 => exit_to_usermode_loop
 => do_syscall_64
 => entry_SYSCALL_64_after_hwframe
 => 0x154b00000002
 => 0x5b40000000a
 => __do_page_fault
 => 0
 => 0x7a84986f29
 => 0x7a84995b76
 => 0x140b00000001
 => 0x5b40000000a
             top-1878    [003] d...  1094.298988: signal_generate: sig=19 errno=0 code=-6 comm=top pid=1878 grp=0 res=0
             top-1878    [003] d...  1094.298992: <stack trace>
 => __send_signal
 => send_signal
 => do_send_sig_info
 => do_send_specific
 => do_tkill
 => __x64_sys_tgkill
 => do_syscall_64
 => entry_SYSCALL_64_after_hwframe
 => 0x85681350000005b4
 => 0x1ffffffff
 => 0xb000012c5
 => 0x85702740000005b4
 => 0x2ffffffff
 => 0xa0000142b
 => 0x85702740000005b4
 => 0xffffffff
             top-1878    [003] d...  1094.298998: signal_deliver: sig=19 errno=0 code=-6 sa_handler=0 sa_flags=0
             top-1878    [003] d...  1094.298999: <stack trace>
 => get_signal
 => do_signal
 => exit_to_usermode_loop
 => do_syscall_64
 => entry_SYSCALL_64_after_hwframe
 => 0x7a8499686b
 => 0x154b00000003
 => 0x5b40000000a
 => _cond_resched
 => 0
 => 0x7a8499672a
 => 0x7a8499690b
 => 0x154500000002
             top-1878    [003] d...  1094.299001: signal_generate: sig=17 errno=0 code=5 comm=bash pid=1348 grp=1 res=0 **부모 프로세스 Bash(1348)에게 17(SIGHLD) 전송 **
             top-1878    [003] d...  1094.299002: <stack trace>
 => __send_signal
 => send_signal
 => do_notify_parent_cldstop
 => do_signal_stop
 => get_signal
 => do_signal
 => exit_to_usermode_loop
 => do_syscall_64
 => entry_SYSCALL_64_after_hwframe
 => 0x84996ce10000007a
 => 0x30000007a
 => 0xa0000140b
 => 0x85868e70000005b4
 => 0xffffffff
 => 0x849969b500000000
 => 0x84996d8b0000007a
 => 0x20000007a
             top-1878    [001] d...  1103.701369: signal_deliver: sig=9 errno=0 code=0 sa_handler=0 sa_flags=0
             top-1878    [001] d...  1103.701380: <stack trace>
 => get_signal
 => do_signal
 => exit_to_usermode_loop
 => do_syscall_64
 => entry_SYSCALL_64_after_hwframe
 => 0
 => 0
 => 0
 => 0
 => 0
 => 0
 => 0
 => 0
             top-1878    [001] dN..  1103.701698: signal_generate: sig=17 errno=0 code=2 comm=bash pid=1348 grp=1 res=0
             top-1878    [001] dN..  1103.701702: <stack trace>
 => __send_signal
 => do_notify_parent
 => do_exit
 => do_group_exit
 => get_signal
 => do_signal
 => exit_to_usermode_loop
 => do_syscall_64
 => entry_SYSCALL_64_after_hwframe
 => 0
 => 0
 => 0
 => 0
 => 0
 => 0
 => 0
 => 0
```

### 리눅스 네트워크(TCP/IP) 동작 분석

- 소켓 open시 fd table에 VFS로 소켓이 생성
- struct file -> struct socket -> struct sock
  - 내부에 write queue, read queue가 있으며 각 element는 struct sk_buff
- 패킷 전송 시 DMA전달 후 인터럽트 발생
  - Top Half: 패킷 수신
  - Bottom Half: soft irq를 통해 데이터 파싱

```plaintext
+--------+------+------+----------+
|Ethernet|  IP  | TCP  |Payload...|
| Header |Header|Header|          |
+--------+------+------+----------+

TCP Control Flags (각 1비트)

URG:    긴급 데이터 플래그 (높은 우선순위)
ACK:    응답 확인 플래그
PSH:    즉시전송 플래그 (Application 계층으로)
RST:     즉시 연결 종료 플래그
SYN:    연결 요청 플래그
FIN:      연결 종료 플래그
```

- MTU(Maximum Transmission Unit): L3 최대 패킷 크기
- MSS(Maximum Segment Size)


### 리눅스 interrupt 처리

- 인터럽트 소스는 IRQ, 내부 타이머 인터럽트
- (Top Half)인터럽트를 처리하기 위한 함수 코드 실행
  - do_IRQ()**인터럽트 벡터에 따라**
    - irq_enter()
    - handle_irq()
    - irq_exit()
- (Bottom Half)나머지 작업은 ksoftirqd를 통해 나중에 실행(soft irq)
  - __do_softirq()
```
    __softirq_pending
     0	HI: 높은 우선순위의 tasklet
     1	TIMER: 타이머
     2	NET_TX: 네트워크 전송
     3	NET_RX: 네트워크 수신
     4	BLOCK: 블록 I/O 완료
     5	IRQ_POLL: 블록 iopoll
     6	TASKLET: 낮은 우선순위의 tasklet
     7	SCHED: 스케줄러의 로드밸런싱
     8	HRTIMER: 고해상도 타이머
     9	RCU: RCU
```
  - tasklet을 통하여 비트별 여러가지 후반부 함수가 추가됨
    - raise_softirq()
      - tasklet_hi_schedule(): Bit0 대응
        - tasklet_hi_vec --> delayed_function --> ...
      - tasklet_schedule(): Bit6 대응
        - tasklet_vec --> delayed_function --> ...

###### 실습. 내 리눅스 interrupt 확인하기
```
# 현재 처리되는 인터럽트 확인하기 
$ cat /proc/interrupts

# 네트워크(enp0s3) 인터럽트 확인하기
# 인터럽트 번호 19번
$ ip link | grep enp0s3
$ cat /proc/interrupts | grep enp0s3
           CPU0       CPU1       CPU2       CPU3       
 19:          0       5187          0      96305   IO-APIC  19-fasteoi   enp0s3  
```
```
# 로컬 타이머 인터럽트 확인하기
# 인터럽트 번호 LOC (0xec: 236번)
$ cat /proc/interrupts | grep "Local timer"
LOC:      58549      72261      76015      70348   Local timer interrupts

# 로컬 타이머 인터럽트 번호(0xec) 확인하기
$ cat arch/x86/include/asm/irq_vectors.h | grep "define LOCAL_TIMER_VECTOR" 
#define LOCAL_TIMER_VECTOR		0xec

# 16진수(0xec)를 10진수(236)로 변환하기
$ echo $((16#ec))
236
```

###### 실습. 인터럽트 모니터링 하기

```
# 리눅스 커널 이벤트 추적준비 tracepoint
$ sudo su
$ cd /sys/kernel/debug/tracing
$ echo 0 > events/enable
$ echo 0 > options/stacktrace
$ echo nop > current_tracer

# 인터럽트 추적하기
$ echo 1 > events/irq/irq_handler_entry/enable
$ echo 1 > events/irq_vectors/local_timer_entry/enable  

# Ctrl + c 로 중단하기
$ cat trace_pipe
```

###### 실습. do_IRQ 루틴 추적하기

```
# 리눅스 커널 이벤트 추적준비 tracepoint
$ sudo su
$ cd /sys/kernel/debug/tracing
$ echo 0 > events/enable
$ echo 0 > options/stacktrace
$ echo nop > current_tracer

# 인터럽트 처리함수 do_IRQ() 추적하기
$ echo 1 > events/irq/irq_handler_entry/enable 
$ echo do_IRQ > set_graph_function
$ echo function_graph > current_tracer


# CPU 선택해서 trace 결과 확인하기
$ cat per_cpu/cpu3/trace
```

###### 실습. softirq 후반부 작업 확인하기

```
# 현재 softirq(인터럽트 후반부작업) 확인하기 
$ cat /proc/softirqs

# 0번째 부터 9번째까지 softirq 확인하기
$ cat /proc/softirqs | grep ":" | nl -v 0

# 타이머(TIMER)와 네트워크 수신(RX) softriq 작업 확인하기 
$ cat /proc/softirqs  | grep TIMER
$ cat /proc/softirqs  | grep RX
```

###### 실습. softirq 커널 모니터링

```
# 리눅스 커널 이벤트 추적준비 tracepoint
$ sudo su
$ cd /sys/kernel/debug/tracing
$ echo 0 > events/enable
$ echo 0 > options/stacktrace
$ echo nop > current_tracer

# softirq(인터럽트 후반부 작업) 추적하기
$ echo 1 > events/irq/softirq_raise/enable 
$ echo 1 > events/irq/softirq_entry/enable   
$ echo 1 > events/irq/softirq_exit/enable 

# Ctrl + c 로 중단하기
$ cat trace_pipe
```

###### 실습. softirq 예약 과정 추적

```bash
# 리눅스 커널 이벤트 추적준비 tracepoint
$ sudo su
$ cd /sys/kernel/debug/tracing
$ echo 0 > events/enable
$ echo 0 > options/stacktrace
$ echo nop > current_tracer

# 인터럽트 처리함수 do_IRQ() 내부에서 softirq(인터럽트 후반부작업) 예약(raise) 과정 추적
$ echo 1 > events/irq/softirq_raise/enable
$ echo do_IRQ > set_graph_function
$ echo function_graph > current_tracer
$ cat trace
...
 3)               |    do_IRQ() {
 3)               |      irq_enter() {
 3) + 11.569 us   |        rcu_irq_enter();
 3) + 35.327 us   |      }
 3)               |      handle_irq() {
...
 3)               |                e1000_intr() {
 3) + 10.552 us   |                  napi_schedule_prep();
 3)               |                  __napi_schedule() {
 3)               |                    __raise_softirq_irqoff() {
 3)               |                      /* softirq_raise: vec=3 [action=NET_RX] */  
```

###### 실습. 인터럽트 후반부 작업 처리 함수 초기화 및 예약

```
# 커널코드에서 tasklet_init() 을 통해서
# 새로운 커널 함수를 인터럽트 후반부 작업 함수로 초기화하는 코드 찾기
$ cd ~/git/linux
$ ag tasklet_init

# iscsi (네트워크 연결 스토리지 I/O 제어 프로토콜)
$ ag tasklet_init | grep completion_tasklet
drivers/scsi/isci/init.c:510:	tasklet_init(&ihost->completion_tasklet,

# 나중에 호출될 미룰 함수 isci_host_completion_routine() 을 tasklet으로 초기화
# 참고: isci_host_alloc() 함수는 modprobe 시 호출된다.
$ vim drivers/scsi/isci/init.c
491 static struct isci_host *isci_host_alloc(struct pci_dev *pdev, int id)
492 {
...
510         tasklet_init(&ihost->completion_tasklet,
511                      isci_host_completion_routine, (unsigned long)ihost); 
```

```
# 커널코드에서 tasklet_schedule() 을 통해서
# tasklet 커널 함수를 인터럽트 후반부 작업 예약(schedule, raise) 하는 지점 찾기
$ cd ~/git/linux
$ ag tasklet_schedule

# iscsi (네트워크 연결 스토리지 I/O 제어 프로토콜)
$ ag tasklet_schedule | grep completion_tasklet
drivers/scsi/isci/host.c:225:		tasklet_schedule(&ihost->completion_tasklet);
drivers/scsi/isci/host.c:615:		tasklet_schedule(&ihost->completion_tasklet);

# tasklet 커널 함수를 인터럽트 후반부 작업 예약(schedule, raise) 하는 코드
# 결과적으로 차후에 softirq 을 통해서 isci_host_completion_routine() 함수가 호출된다.
$ vim drivers/scsi/isci/host.c
 608 irqreturn_t isci_intx_isr(int vec, void *data)
 609 {
...
 615                 tasklet_schedule(&ihost->completion_tasklet);
```

### workqueue

- 인터럽트 / 시스템콜 / 예외처리 등을 위한 다양한 커널 함수들을 한꺼번에 전부 호출하지 않고 후반부로 미뤄서 나중에 실행할 수 있도록 하는 기술
- 실행해야 할 커널함수 중 늦게 처리해도 되는 함수를 workqueue에 저장
- workqueues -1...n-> workqueue_struct -> pool_workqueue -> delayed_works
- CPU{X} -1...n-> worker_pool -1...n-> worker("kworker/0:1"..., worklist)
- queue_work함수를 통해 workqueue_struct에 enqueue하며 다음의 동작 중 하나를 실행
  - worker의 worklist에 enqueue
  - pool_workqueue의 delayed_works에 enqueue

###### 실습. workqueue 함수예약(queue_work)추적

```
# 리눅스 커널 이벤트 추적준비 tracepoint
$ sudo su
$ cd /sys/kernel/debug/tracing
$ echo 0 > events/enable
$ echo 0 > options/stacktrace
$ echo nop > current_tracer

# workqueue(후반부 작업) 처리 되는 과정 추적하기
$ echo 1 > events/workqueue/workqueue_queue_work/enable
$ echo 1 > options/stacktrace
$ cat trace
bash-1709  [003] d... 15710.528734: workqueue_queue_work: work struct=000000006ed0833c 
function=flush_to_ldisc workqueue=00000000e3baf7b4 req_cpu=64 cpu=3
bash-1709  [003] d... 15710.528754: <stack trace>
 => trace_event_raw_event_workqueue_queue_work
 => __queue_work
 => queue_work_on
 => pty_write
 => do_output_char
 => n_tty_write
 => tty_write
 => vfs_write
 => ksys_write
 => do_syscall_64
```

###### 실습. workqueue 함수실행(execute_start) 추적
```
# 리눅스 커널 이벤트 추적준비 tracepoint
$ sudo su
$ cd /sys/kernel/debug/tracing
$ echo 0 > events/enable
$ echo 0 > options/stacktrace
$ echo nop > current_tracer

# workqueue(후반부 작업) 실제호출되는 과정 추적하기
$ echo 1 > events/workqueue/workqueue_execute_start/enable
$ cat trace
...
kworker/u8:3-4284  [003] .... 16111.011605: workqueue_execute_start: work struct 000000006ed0833c:  
function flush_to_ldisc
...

```


### BPF

- 리눅스 커널에서 동적으로 바이너리 코드를 실행하는 엔진
- readelf시 Linux BPF로 출력됨
- 안전성 보장? --> BPF verifier가 안전 보장
- 활용 시나리오
  - LOAD
  - ATTACH
  - CALLBACK

###### 실습. xdp_rxq_info 테스트

```bash
$ cd ~/git/linux/sample/bpf
$ sudo ./xdp_rxq_info --dev lo
$ ping 127.0.0.1
```

###### 실습. xdp-drop.c

```c
#include <linux/bpf.h>
#ifndef __section
# define __section(NAME)                  \
  __attribute__((section(NAME), used))
#endif

__section("prog")
int xdp_drop(struct xdp_md *ctx)
{
   return XDP_DROP;
}

char __license[] __section("license") = "GPL";
```
```bash
$ clang -O2 -Wall -target bpf -I/usr/include/x86_64-linux-gnu/ -c xdp-drop.c -o xdp-drop.o
$ sudo ip link set dev lo xdp obj xdp-drop.o
```

```bash
# 오류 시
$ cd ~/git
$ git clone git://git.kernel.org/pub/scm/network/iproute2/iproute2.git
$ cd iproute2

# 패키지 확인 후 설치
$ ./configure
$ sudo apt install -y libdb-dev libdb5.3-dev libmnl-dev libselinux1-dev
$ make && sudo make install

# BPF 주입 시도
$ sudo ip link set dev lo xdp obj xdp-drop.o
```

```bash
#xdp 제거
$ sudo ip link set dev lo xdp off
$ ip a
```

###### 실습. BPF에서 패킷 보기

```bash
# BPF 측정하고
$ cd ~/git/linux/samples/bpf/  
$ sudo ./tracex1

# 다른터미널 ping 테스트
$ ping 127.0.0.1
```
