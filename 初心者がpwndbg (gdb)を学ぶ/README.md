https://qiita.com/0108mitaka/items/4e601fd4cb9837bba5f6

# 初心者が pwndbg (gdb)を学ぶ

pwndbg とは、gdb を便利にしてくれる拡張スクリプトです。
gdb とは、主に gcc でコンパイルしたプログラムをデバッグするツールです(ざっくり)。
pwndbg は CTFer がよく使っているイメージがある。

この記事では、これ以上ない程単純な reversing の問題を、勉強のためにあえて pwndbg 縛りで解きます

リバースエンジニアリングは、自作や CTF など許可されたファイル以外にはしないようにしましょう(そんなレベルの記事ではないですが、念のため)。

## 環境

WSL2 (win10)
Ubuntu 20.04.4 LTS

pwndbg はこれ。
https://github.com/pwndbg/pwndbg
検索する時 dbg と gdb を間違えやすいので注意(1 敗)

`gdb`とコマンドをうつことで、pwndbg を開ける。
(pwndbg の文字が出ていないならインストールに失敗してます)
動作が確認出来たら、一旦`q`で閉じます。

## ターゲット

ターゲットはこちら。

```c:tar1.c
#include <stdio.h>
void main(){
    int a,b,c;
    a=2;
    b=3;
    c = a * b;
    if (c == 20){
        printf("C is %d\n",c);
        printf("Awesome!\n");
    }
    else{
        printf("C is %d\n",c);
    }
}
```

```
user@WSL$ gcc tar1.c -o tar1
user@WSL$ ./tar1
C is 6
```

当然 c は 6 になり、Awesome！とは表示されない。ので、Awesome を出させます。
なお CTF などではこんなソースコードは与えられません。pwndbg を使う前に Ghidra などの解析ソフトでデコンパイルし、到達したい場所のある程度の目星を付ける必要があります。

pwndbg で開いてみる。

```
user@WSL$ gdb -q tar1
pwndbg> start
```

pwndbg が色々ログを出してくれます。
start コマンドでは、main や init などの丁度いい所で処理を一旦ストップしてくれます (便利！)
ちゃんと説明すると、丁度いい所にブレークポイントを貼っています。

```
pwndbg> start
Temporary breakpoint 1 at 0x1169

Temporary breakpoint 1, 0x0000555555555169 in main ()
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
─────────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]─────────────────────────────────
*RAX  0x555555555169 (main) ◂— endbr64
*RBX  0x5555555551d0 (__libc_csu_init) ◂— endbr64
*RCX  0x5555555551d0 (__libc_csu_init) ◂— endbr64
*RDX  0x7fffffffdd18 —▸ 0x7fffffffdf5c ◂— 'SHELL=/bin/bash'
*RDI  0x1
*RSI  0x7fffffffdd08 —▸ 0x7fffffffdf3f ◂— '/{path}/tar1'
 R8   0x0
*R9   0x7ffff7fe0d60 (_dl_fini) ◂— endbr64
*R10  0x7
*R11  0x2
*R12  0x555555555080 (_start) ◂— endbr64
*R13  0x7fffffffdd00 ◂— 0x1
 R14  0x0
 R15  0x0
 RBP  0x0
*RSP  0x7fffffffdc18 —▸ 0x7ffff7deb083 (__libc_start_main+243) ◂— mov edi, eax
*RIP  0x555555555169 (main) ◂— endbr64
──────────────────────────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────────────────────────
 ► 0x555555555169 <main>       endbr64
   0x55555555516d <main+4>     push   rbp
   0x55555555516e <main+5>     mov    rbp, rsp
   0x555555555171 <main+8>     sub    rsp, 0x10
   0x555555555175 <main+12>    mov    dword ptr [rbp - 0xc], 2
   0x55555555517c <main+19>    mov    dword ptr [rbp - 8], 3
   0x555555555183 <main+26>    mov    eax, dword ptr [rbp - 0xc]
   0x555555555186 <main+29>    imul   eax, dword ptr [rbp - 8]
   0x55555555518a <main+33>    mov    dword ptr [rbp - 4], eax
   0x55555555518d <main+36>    cmp    dword ptr [rbp - 4], 0x14
   0x555555555191 <main+40>    jne    main+78                <main+78>
───────────────────────────────────────────────────────[ STACK ]────────────────────────────────────────────────────────
00:0000│ rsp 0x7fffffffdc18 —▸ 0x7ffff7deb083 (__libc_start_main+243) ◂— mov edi, eax
01:0008│     0x7fffffffdc20 —▸ 0x7ffff7ffc620 (_rtld_global_ro) ◂— 0x50a6600000000
02:0010│     0x7fffffffdc28 —▸ 0x7fffffffdd08 —▸ 0x7fffffffdf3f ◂— '/{path}/tar1'
03:0018│     0x7fffffffdc30 ◂— 0x100000000
04:0020│     0x7fffffffdc38 —▸ 0x555555555169 (main) ◂— endbr64
05:0028│     0x7fffffffdc40 —▸ 0x5555555551d0 (__libc_csu_init) ◂— endbr64
06:0030│     0x7fffffffdc48 ◂— 0xaf2eec1db698b8c3
07:0038│     0x7fffffffdc50 —▸ 0x555555555080 (_start) ◂— endbr64
─────────────────────────────────────────────────────[ BACKTRACE ]──────────────────────────────────────────────────────
 ► f 0   0x555555555169 main
   f 1   0x7ffff7deb083 __libc_start_main+243
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
pwndbg>
```

出ているのはスタックの中身やアセンブリ言語など。今見るのはここ！
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/609800/ea8f8a99-abec-3730-7132-e770ee464835.png)

目的の if 文はアセンブリ言語で`cmp`なので、直前まで移動する。

```
pwndbg> n
```

をするごとに 1 行ずつ処理されるので、`n`を連打。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/609800/369836d7-e280-3977-0aa5-a8e318bb7bdb.png)
`cmp`の直前まで来た。
ここで、

```
pwndbg> i r
```

を入力すると
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/609800/edcc730f-f0d9-a530-fd15-059a0e0b782a.png)
現在のレジスタの情報が見れます。便利！
先ほどのアセンブリ言語から`eax`と`0x14`を比較して分岐していることが分かるので、レジスタ`rax`が`0x14`であれば望み通りの分岐に行ってくれる。

```
pwndbg> set $rax=0x14
```

を入力し、もう一度`i r`で確認する。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/609800/24dc9cd0-d6bf-b7f5-54d6-e62a1ed700d4.png)
レジスタの中身の書き換えに成功！

```
pwndbg> c
```

でそのまま最後まで一気に実行。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/609800/f41129b4-6276-bf57-5e7f-3bb3cbef69ef.png)
無事(？)、6 となるはずの c が 20 に書き換えられ、出力されるはずのないコードが出力されました！

......

ところで、レジスタの中身を書き換えられるのなら、比較した結果の True か False を直接書き換えればいいと思いませんか？
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/609800/fce788dc-4fd1-0d95-fe40-e196d77f7088.png)
最初から新しい状態で`start`し、cmp の後まで来ました。この状態で、レジスタの中身は次の通りです。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/609800/f435c817-0d98-a400-c171-a45fa43de834.png)
直前の cmp の結果は eflags に格納されています。eflags は各ビットがフラグとして扱われているレジスタです。
計算機の勉強をしたことがある人なら、桁上がりで CF(キャリーフラグ)がセットされる、などと聞いたことがあるのでは。

重要なのは ZF(ゼロフラグ)です。これは 6 番目のフラグです。
直前の操作の結果が 0 になったときだけセットされます。
cmp 命令の中身は実質引き算で、差が 0 であれば等しいとされるのです。

というわけで、eflags の 6 番目を書き換えて ZF を 1 にしてしまいましょう。

```
pwndbg> set $eflags |= (1<<6)
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/609800/e7219835-1a3c-711e-5316-946315aaa4bc.png)
無事 ZF が 1 になりました。
`c`で最後まで行きます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/609800/0d42c50e-74c3-7a93-7433-0500ab5ff91c.png)
c が 20 ではなく 6 のままなのに、if(c==20)を通過してしまいました。

## おわり

とても便利……
普通にデバッガとして使う場合でも超便利。

このツール、まだまだ色んなことができます。
~~(ブレークポイントを使わない gdb 記事があっていいのか)~~
プログラムの任意の処理に対応するアドレスを特定すれば、そこにブレークポイントを貼って処理を開始することでブレークポイントで停止し、レジスタの中身などを確認することができます(効率的！)

おわり
