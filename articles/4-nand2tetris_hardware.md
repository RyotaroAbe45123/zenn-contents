---
title: "Nand2Tetris ハードウェア編"
emoji: "🖥️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CPU", "nand2tetris", "hardware", "computer_architecture"]
published: true
---

# Nand2Tetris ハードウェア編: 1~5章の実装とハマりどころ

## この記事の概要
本記事では、Nand2Tetris のハードウェア部分に焦点を当て、基本的な NAND ゲートから始めて、コンピュータの主要なコンポーネントをどのように構築するかを解説します。特に **1~5章の実装過程** を解説し、筆者が躓いたポイントや注意点を共有します。

- **対象読者**
  - 非情報系出身者
  - 仕事や趣味でソフトウェア開発はしているが、いつかじっくりコンピュータの仕組みを勉強したいなと思っていた方
  - 手を動かしながら学ぶのが好きな方


本書を通じて作成したプログラムは、GitHub に公開しています。
https://github.com/RyotaroAbe45123/nand2tetris

## Nand2Tetris とは？
[Nand2Tetris](https://www.nand2tetris.org/) は、NANDゲートからスタートし、徐々にコンピュータおよびそのうえで動作するアプリケーションを作成する学習プロジェクトです。
上記はウェブサイト上でプロジェクトが公開されており、それとは別に日本語書籍も出版されています。
https://www.oreilly.co.jp/books/9784873117126/

第2版が2021年に出版され、その邦訳版が2024年に発売されました。
その際の、翻訳を斎藤康毅さんが担当されました。
この方は、[ゼロから作るDeep Learning](https://www.oreilly.co.jp/books/9784873117584/)の著者でもあります。
私も昔、この本で手を動かしながら勉強させてもらった記憶があり、それが大変わかりやすかったので、今回Nand2Tetrisにチャレンジすることにしました。
https://x.com/SaitohKoki/status/1863373192133476410

このプロジェクトの **ハードウェア編（1~5章）** では、以下のような流れでコンピュータを構築していきます。

## 1~5章のアジェンダ
| 章 | 内容 | 実装するもの |
|----|------|------------|
| 1章 | ブール論理 | AND・OR・NOTなどの基本ゲート |
| 2章 | ブール演算 | 加算器・インクリメンタなどから16bitALU |
| 3章 | メモリ | 実装済みDフリップフロップを用いて、1bitレジスタから16KRAM |
| 4章 | 機械語 | Hackアセンブリで加算、乗算、I/O |
| 5章 | コンピュータアーキテクチャ | これまで作成したMemory, CPUなどからHackコンピュータを |

以下の図のように、コンピュータシステムは大きくハードウェアとソフトウェアに分けられます。(各番号が各章と対応している)
よって、1章でNANDから一通りの基本ゲートを作成し、2, 3章でALUやメモリのチップを作成、4章のHack機械語に沿って、5章でこれまで作成してきたパーツを用いてHackコンピュータを作成する流れです。

![overall](/images/4/01_overall.jpg =500x)
*図Ⅰ-1より引用：コンピュータシステムの主要なモジュール*

---

## 1章: ブール論理
### ゴール：基本ゲートの実装
1章では、真理値表から基本ゲートをハードウェア記述言語 (Hardware Description Language: HDL) で記述し、シミュレータで動作検証を行います。
ここで作成するのは、AND, AND16bit, DMux, DMux4way, Dmux8way, Mux, Mux16bit, Mux4way16bit, Mux8way16bit, Not, Not16bit, Or, Or16bit, Or8way, Or16bit です。
例として、ANDゲートの真理値表とHDLを記載します。
| a | b | out |
|---|---|-----|
| 0 | 0 | 0 |
| 0 | 0 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |
```hdl
CHIP And {
    IN a, b;
    OUT out;
    PARTS:
    Nand(a=a, b=b, out=nandOut);
    Nand(a=nandOut, b=nandOut, out=out);
}
```
上記HDLをシミュレータで実行し、テストケースを通すことで、ANDゲートが正しく動作することを確認します。この検証を同様にほかの基本ゲートについても実施します。
個人的には、マルチプレクサ (Multiplexor: Mux) の条件分岐を論理回路で設計するのが難しくハマりどころでした。
:::details Muxの実装
```hdl
CHIP Mux
    IN a, b, sel;
    OUT out;

    PARTS:
    // if sel then a, else b
    Not(in=sel, out=notsel);
    And(a=notsel, b=a, out=out1);
    And(a=sel, b=b, out=out2);
    Or(a=out1, b=out2, out=out);
```
:::

HDLを実装したら、ハードウェアシミュレータを起動して、テストケースを実行します。
ハードウェアを起動すると左の画面が立ち上がり、それにHDLとテストスクリプトを読み込むと右のようになります。
あとは左から2,3番目のボタンからテストを実行し、**End of script - Comparision ended successfully**が表示されればOKです。
| ![](/images/4/02_simulator.png =500x) | ![](/images/4/03_simulator2.png =500x) |
| ----------------------------------------- | ----------------------------------------- |


## 2章: ブール演算
### ゴール：ALUの実装
2章では、中央演算装置 (CPU) の中核である算術論理演算装置 (ALU) を実装します。
半加算器、全加算器、16bit加算器と実装して、ALU, インクリメンタの順になります。

ALUの実装は以下の真理値表に沿って、2つの入力と6つの制御ビットを受け取って、out, zr, ng を出力します。zr, ngはこの段階では何に使うのがわかりませんが、5章のCPU実装で使います。
:::details ALUの真理値表
| zx | nx | zy | ny | f | no | out |
| -- | -- | -- | -- | -- | -- | -- |
|  1 |  0 |  1 |  0 |  1 |  0 |  0 |
|  1 |  1 |  1 |  1 |  1 |  1 |  1 |
|  1 |  1 |  1 |  0 |  1 |  0 |  -1 |
|  0 |  0 |  1 |  1 |  0 |  0 |  x |
|  1 |  1 |  0 |  0 |  0 |  0 |  y |
|  0 |  0 |  1 |  1 |  0 |  1 |  !x |
|  1 |  1 |  1 |  0 |  0 |  1 |  !y |
|  0 |  0 |  1 |  1 |  1 |  1 |  -x |
|  1 |  1 |  0 |  0 |  1 |  1 |  -y |
|  0 |  1 |  1 |  1 |  1 |  1 |  x+1 |
|  1 |  1 |  0 |  1 |  1 |  1 |  y+1 |
|  0 |  0 |  1 |  1 |  1 |  0 |  x-1 |
|  1 |  1 |  0 |  0 |  1 |  0 |  y-1 |
|  0 |  0 |  0 |  0 |  1 |  0 |  x+y |
|  0 |  1 |  0 |  0 |  1 |  1 |  x-y |
|  0 |  0 |  0 |  1 |  1 |  1 |  y-x |
|  0 |  0 |  0 |  0 |  0 |  0 |  x&y |
|  0 |  1 |  0 |  1 |  0 |  1 |  x|y |
:::
ALUの条件分岐、特にzr, ngの出力をどうするかが難しいですが、それ以外は指示に従い、スムーズに実装ができると思います。
前者はx,yの①等号判定、後者はoutの②符号判定です。
①等号判定にはXORゲートを使いました。本当はXNORを用いるともっとスムーズに実装できたかと思いますが、1章で実装していなかったので、XORにしました。（たぶんもっとよい実装方法があると思いますのでぜひコメントいただければと思います）
②符号判定には、outの最上位ビットを取り出して、ngに代入しました。
参考までに筆者が実装したALUは[こちら](https://github.com/RyotaroAbe45123/nand2tetris/blob/main/02_BooleanArithmetic/ALU.hdl)になります。


また、インクリメンタを加算器を用いて実装しますが、こちらは、3章のプログラムカウンタ (PC) で用います。

## 3章: メモリ
### ゴール：RAMの実装
これまでは組み合わせ回路だけでしたが、この章から順序回路が登場します。
データフリップフロップ (Data Flip Flop: DFF) のフィードバックループによって、状態を維持することができるようになります。
DFFを最小構成として、1bitレジスタ、16bitレジスタ、16bitランダムアクセスメモリ (Random Access Memory: RAM) を実装します。
最後にCPUで必要になるプログラムカウンタ (Program Counter: PC) を実装します。

まずは、1bitレジスタの実装をします。レジスタは、loadビットが1のときにinの値を保持し、0のときに保持しません。
この条件分岐は1章で実装したMuxが使えます。また、DFFを用いて、レジスタの状態を保持します。DFFの実装はビルトインチップとして提供されているので、そのまま使えます。
同様に、1bitRegisterを16bitRegisterに拡張し、16bitRAMを実装します。
また、それらを並列にすることで、16KRAMを実装します。
![overall](/images/4/04_register.png =500x)
*Registerの論理回路*
```hdl
CHIP 1BitRegister
    IN in, load;
    OUT out;

    PARTS:
    Mux(a=backward, b=in, sel=load, out=forward);
    DFF(in=forward, out=out, out=backward);
```

私はこの章では、PCの実装にドハマリしました。(めちゃくちゃ時間を溶かしました)
PCは入力、出力、loadビット、incビット、resetビットから構成されます。
resetビットが1のときはoutは0になり、incビットが1のときはoutに1を加えます。loadビットが1のときはinの値を保持します。
これらの条件分岐の優先度付けに苦戦しました。
reset > load > incと順番は決められているので、どうやったらこの順番を実装できるかが難しかったです。
優先度が高いとは、論理回路上の入力側か出力側かを考えるとわかりやすいです。分岐にはMuxが使えます。
:::details PCの実装(参考までに)
```hdl
CHIP PC
    IN in[16],inc, load, reset;
    OUT out[16];
    
    PARTS:
    Inc16(in=backward, out=incout);
    Mux16(a=backward, b=incout, sel=inc, out=incforward);
    Mux16(a=incforward, b=in, sel=load, out=loadforward);
    Mux16(a=loadforward, b=false, sel=reset, out=resetforward);
    // ここはDFF16であればよいのだが、実装がなく面倒なので、Registerを代用する。このときloadには常に1を入力する。
    Register(in=resetforward, load=true, out=out, out=backward);
```
:::

## 4章: 機械語
### ゴール：アセンブリの作成
4章で突然機械語の話が出てきます。あれ？CPU作るんじゃないの？と思うかもしれませんが、CPUを作るためには、命令がどの順番でどこにアクセスして実行されるのかを理解しておくととてもイメージが湧きやすいです。
なので、先んじて、機械語およびアセンブリの作成を行い、その動作イメージを沸かせているのだと解釈しました。
実際、4章の段階ではわからないですが、5章のCPU実装の段階で、とても役に立ちました。またそこから4章に戻るとなお理解が深まります。

ここでは、アドレスレジスタ、データレジスタ、データメモリ、命令メモリ、アドレス命令 (A命令)、計算命令 (C命令) を駆使して乗算のプログラムを実装します。
いつも高級言語でfor文とかwhileループとか適当に使っているのが、ここでは、本当にひとつずつの処理を順番にしか実行できません。一方で完全にコントロールしている感じがして、とても楽しいです。

以下に加算プログラムのアセンブリ例を示します。
これは、RAM[2] = RAM[0] + RAM[1] + 17 を実行するプログラムです。
処理のイメージが沸かない場合は、ひとつずつ処理内容を記載して動作を照らし合わせると理解しやすいです。
```asm
    @R0 -> RAM[0]にアクセス
    D=M -> データレジスタにRAM[0]から読み取った値を格納
    @R1 -> RAM[1]にアクセス
    D=D+M -> データレジスタにRAM[1]から読み取った値を既存の値に加算
    @17 -> RAM[17]にアクセス
    D=D+A -> データレジスタに17を加算
    @R2 -> RAM[2]にアクセス
    M=D -> RAM[2]にデータレジスタの値を格納
(END)
    @END
    0;JMP
```

アセンブリを実装したら、CPUエミュレータを起動して、テストケースを実行します。
アセンブラは実装せずとも上記エミュレータ上でアセンブリを実行できます。
ここで1つずつ実行ボタンを押下していくと、どの順番で命令が実行されているのかを追うことができるのでとてもわかりやすいです。（目で追うだけでも楽しい。想定通りに動作するとめっちゃうれしい。）
| ![](/images/4/05_cpuemulator.png =500x) | ![](/images/4/06_cpuemulator2.png =500x) |
| ----------------------------------------- | ----------------------------------------- |

## 5章: コンピュータアーキテクチャ
### Hackコンピュータの実装
ここまでで作成したものを活用し、CPU, Memoryを作成し、それらを組み合わせて、Hackコンピュータを構築します。
Hackコンピュータはこれらをシンプルに組み合わせたものなので、あまりつまりどころはないと思います。
```hdl
CHIP Computer
    IN reset;

    PARTS:
    ROM32K(address=pc, out=instruction);
    CPU(
        inM=inM, instruction=instruction, reset=reset,
        outM=outM, writeM=writeM, addressM=addressM, pc=pc
    );
    Memory(in=outM, load=writeM, address=addressM, out=inM);
```

一方で、CPUの実装が最も難しかったです。
3章で作成したパーツや4章で実装したアセンブリの仕様から、CPUの動作イメージは湧くと思います。
それらを16bitのinstructionの入力に合わせて、データレジスタ、アドレスレジスタ、ALU、PCに接続していきます。
この分岐が複雑で、テストを通すまでに何度も修正しました、、、
ひとつずつ仕様と照らし合わせると時間がかかるとは思いますが、テストを通すことはできると思います。
![cpu](/images/4/07_cpu.jpg =750x)
*図5-8より引用：Hack CPUの実装案*

以下にCPUのHDLを記載します。注意点としては、Registerの使い回しではなく、ARegister, DRegisterを使うことです。
また、jump判定の部分の条件分岐が複雑ですが、4章で用いたHack命令セットとjjjビット(ジャンプ命令), zrビット, ngビットを照らし合わせることで、実装できると思います。ここでのzr, ngはALUの出力を参照しています。(ここで使うのかと思いました！)
```hdl
CHIP CPU {

    IN  inM[16],         // M value input  (M = contents of RAM[A])
        instruction[16], // Instruction for execution
        reset;           // Signals whether to re-start the current
                         // program (reset==1) or continue executing
                         // the current program (reset==0).

    OUT outM[16],        // M value output
        writeM,          // Write to M? 
        addressM[15],    // Address in data memory (of M)
        pc[15];          // address of next instruction

    PARTS:
    // 計算結果をAレジスタに格納するかどうか
    // dddの最上位ビットが1のとき、destにAが含まれるので、その判断をMuxで実施
    Mux16(a=instruction, b=aluOut, sel=instruction[15], out=forward1);
    // アドレスレジスタにA命令を格納するかどうか。最上位ビットが1ならA命令なので、格納する。
    Not(in=instruction[15], out=aInstructionFlag);
    // C命令かつdestにAが含まれる場合も
    And(a=instruction[15], b=instruction[5], out=cInstructionAndAInDest);
    Or(a=aInstructionFlag, b=cInstructionAndAInDest, out=aRegisterLoad);
    ARegister(
        in=forward1, load=aRegisterLoad,
        out=aRegisterOut, out[0..14]=addressM
    );
    // ALUの入力がアドレスレジスタかデータメモリからかを判定
    Mux16(a=aRegisterOut, b=inM, sel=instruction[12], out=yToALU);

    DRegister(in=aluOut, load=instruction[4], out=xToALU);
    ALU(
        x=xToALU, y=yToALU,
        zx=instruction[11], nx=instruction[10],
        zy=instruction[9], ny=instruction[8],
        f=instruction[7], no=instruction[6],
        out=aluOut, out=outM,
        zr=zr, ng=ng
    );
    And(a=instruction[15], b=instruction[3], out=writeM);

    PC(
        in=aRegisterOut,
        load=pcLoadFlag, inc=pcIncFlag, reset=reset,
        out[0..14]=pc
    );
    // jump判定
    Not(in=zr, out=notZr);
    Not(in=ng, out=notNg);
    And(a=instruction[2], b=instruction[1], out=out1);
    And(a=instruction[0], b=out1, out=out11);
    And(a=zr, b=instruction[1], out=out2);
    And(a=ng, b=instruction[2], out=out3);
    And(a=notZr, b=notNg, out=out4);
    And(a=instruction[0], b=out4, out=out44);
    Or(a=out11, b=out2, out=out12);
    Or(a=out3, b=out44, out=out34);
    Or(a=out12, b=out34, out=pcLoad);
    And(a=pcLoad, b=instruction[15], out=pcLoadFlag);
    Not(in=pcLoadFlag, out=pcIncFlag);
}
```

## まとめ
1~5章を通じて、Nand2Tetrisのハードウェア編を実装しました。これにより、Hack機械語の動作するHackコンピュータをNANDゲートから構築することができました。
NANDゲートから手を動かしながら実装することで、コンピュータの基本的な仕組みを理解することができました。
また、シンプルに構築したものが動作するのを見られると楽しいですねぇ！
次の6章ではアセンブラの実装になります。こちらは別途記事にまとめたいと思います。
