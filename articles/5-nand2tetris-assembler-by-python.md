---
title: "Nand2Tetrisのアセンブラ(6章)をPythonで実装する"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nand2tetris", "assembly", "python"]
published: true
---

# Nand2TetrisのアセンブラをPythonで実装する

## この記事の概要
本記事では、Nand2Tetrisにおいて、アセンブラをPythonで実装する方法を解説します。
なお、1~5章のハードウェアはこちらを参考にしてください。
https://zenn.dev/ryotaro45123/articles/4-nand2tetris_hardware

- **対象読者**
  - 普段python使っている人
  - Nand2Tetris進めている方
  - 手を動かしながら学ぶのが好きな方

本書を通じて作成したプログラムは、GitHub に公開しています。
https://github.com/RyotaroAbe45123/nand2tetris

また、記事を作成するうえで、こちらをとても参考にさせていただきました。
https://zenn.dev/yukiyada/articles/44805448111905

## アセンブラとは？
アセンブラとは、アセンブリ(シンボルを用いた少し人間が読みやすくしたもの)を機械語(バイナリ)へ変換するプログラムです。
特に、Hackコンピュータにおいては、Hackアセンブラは*Hackアセンブリ*を*Hack機械語*へ変換します。
具体的には、図1に示す記号を用いた表現を**Hackアセンブリ**と呼びます。
一方、図2に示す0,1のみのバイナリーコードを**Hack機械語**と呼びます。
こちらの2つは対応しており、図1のコメントに示す動作をHack機械語で実行することができます。
また、4章で実施したアセンブリのとおり、Hackアセンブリでは、@から始まるA命令とC命令が交互に来ており、それぞれに対応するように、バイナリーコードでも先頭が0の行と1の行が交互に記載されます。
![assembly](/images/5/01_assembly.png =450x)
*図1 Hackアセンブリ*
![binary](/images/5/02_binary.png =300x)
*図2 Hack機械語*


## PythonでHackアセンブラを実装する
### 方針
まずは、シンボルが含まれないアセンブリを変換することができる**基本版アセンブラ**を実装します。
その後、シンボルも含まれた**完全版アセンブラ**を実装します。
両者ともアセンブリが記載されたファイルを読み込み、上から順番に図3に示す命令セットに沿って、バイナリーコードへ変換する。
一方、完全版アセンブラは、シンボルも対応するため、シンボルテーブルを作成する点が違います。
それぞれの実装については、以下に詳細を記載します。
![design](/images/5/03_design.png =500x)
*図3 Hack命令セット(図6-2より引用)*

### 基本版アセンブラ
基本アセンブラでは、A命令にシンボルが含まれません。つまり、@xxxのxxxは必ず10進数となります。
したがって、A命令は10進数を16bit2進数へ変換、C命令は図3に沿って変換となります。
必要なモジュールはParser, Code, Hackの3つです。
Parserクラスでは、入力を解析して一連の命令の行にします。
Codeクラスでは、各行をバイナリーコードへ変換します。
Hackクラスでは、ParserとCodeを用いて、アセンブリをバイナリーコードへ変換します。
```python:Parserクラス
class Parser:
    def __init__(self, asm_file_path: str) -> None:
        """
        入力ファイルを受け付ける
        """
        with open(asm_file_path, "r") as fp:
            self.asm = fp.read()
            self.asm = self.asm.split("\n")

        self.order = None
    
    def has_more_lines(self) -> bool:
        """
        入力にまだ行があるか判断する
        """
        return True if self.asm else False

    def advance(self) -> None:
        """
        入力から次の命令を読み込み、それを現在の命令にする
        """
        self.order = self.asm.pop(0)
        if self.order.startswith("//") or self.order == "":
            self.order = None
    
    def instruction_type(self) -> Instruction:
        """
        現在の命令のタイプを返す
        """
        if self.order.startswith("@"):
            return Instruction.A
        elif self.order.startswith("(") and self.order.endswith(")"):
            return Instruction.L
        else:
            return Instruction.C
    
    def symbol(self) -> str:
        """
        現在の命令に、シンボルが含まれる場合は、シンボルを
        そうでない場合は、10進数を文字列で返す
        """
        if self.instruction_type == Instruction.L:
            return self.order[1:-1]
        else:
            return self.order[1:]
    
    def dest(self) -> str:
        """
        現在のC命令のdestを返す
        """
        if "=" in self.order:
            return self.order.split("=")[0]
        else:
            return "null"
    
    def comp(self) -> str:
        """
        現在のC命令のcompを返す
        """
        if "=" in self.order and ";" in self.order:
            return self.order.split("=")[-1].split(";")[0]
        elif "=" in self.order:
            return self.order.split("=")[-1]
        elif ";" in self.order:
            return self.order.split(";")[0]

    def jump(self) -> str:
        """
        現在のC命令のjumpを返す
        """
        if ";" in self.order:
            return self.order.split(";")[-1]
        else:
            return "null"
```
```python:Codeクラス
class Code:
    @classmethod
    def dest(self, arg: str) -> str:
        """
        destニーモニックのバイナリーコード(3ビット)を返す
        """
        is_m_in_dest = "1" if "M" in arg else "0"
        is_d_in_dest = "1" if "D" in arg else "0"
        is_a_in_dest = "1" if "A" in arg else "0"
        return is_a_in_dest + is_d_in_dest + is_m_in_dest

    @classmethod
    def comp(self, arg: str) -> str:
        """
        compニーモニックのバイナリーコード(7ビット)を返す
        """
        unified_arg = arg.replace("M", "A")
        comp_map = {
            "0": "101010",
            "1": "111111",
            "-1": "111010",
            "D": "001100",
            "A": "110000",
            "!D": "001101",
            "!A": "110001",
            "-D": "001111",
            "-A": "110011",
            "D+1": "011111",
            "A+1": "110111",
            "D-1": "001110",
            "A-1": "110010",
            "D+A": "000010",
            "D-A": "010011",
            "A-D": "000111",
            "D&A": "000000",
            "D|A": "010101",
        }
        return comp_map[unified_arg]    

    @classmethod
    def jump(self, arg: str) -> str:
        """
        jumpニーモニックのバイナリーコード(3ビット)を返す
        """
        jump_map = {
            "null": "000",
            "JGT": "001",
            "JEQ": "010",
            "JGE": "011",
            "JLT": "100",
            "JNE": "101",
            "JLE": "110",
            "JMP": "111",
        }
        return jump_map[arg]
```
```python
class Hack:
    def __init__(self, asm_file_name: str) -> None:
        self.asm_file_path = f"./{asm_file_name}"
        self.parser = Parser(asm_file_path=self.asm_file_path)
        # 出力ファイルのProg.hackを作成する
        self.hack_file_path = f"./{asm_file_name.replace('asm', 'hack')}"
        with open(self.hack_file_path, "w") as fp:
            fp.write("")
            
    def do_binary_conversion(self) -> None:
        """
        バイナリーコードへ変換
        """
        self.convert_asm_to_hack()

    def convert_asm_to_hack(self) -> None:
        # 各行(アセンブリ命令)を反復処理する
        # C命令については、各フィールドをバイナリーコードに変換して、連結する
        # A命令については、xxxをバイナリーコードに変換する
        while self.parser.has_more_lines():
            self.parser.advance()
            if self.parser.order is None:
                continue

            binary_code = ""
            if self.parser.instruction_type() == Instruction.A:
                symbol_int = int(self.parser.symbol())
                binary_16bit_str = f"{symbol_int:016b}"
                binary_code += binary_16bit_str
            elif self.parser.instruction_type() == Instruction.C:
                dest_asm = self.parser.dest()
                comp_asm = self.parser.comp()
                jump_asm = self.parser.jump()
                
                dest_binary = Code.dest(dest_asm)
                comp_binary = Code.comp(comp_asm)
                jump_binary = Code.jump(jump_asm)
                
                binary_code += "111"
                is_a = "1" if "M" in comp_asm else "0"
                binary_code += is_a
                binary_code += comp_binary
                binary_code += dest_binary
                binary_code += jump_binary
            # elif parser.instruction_type() == Instruction.L:
            #     pass

            with open(self.hack_file_path, "a") as fp:
                fp.write(binary_code)
                fp.write("\n")
```

### 完全版アセンブラ
完全版アセンブラでは、A命令にシンボルが含まれる点が異なります。そのため、まずはシンボルを解析する必要があります。
よって、まずは命令を1周し、各シンボルに対応するアドレスをシンボルテーブルに格納します。(第1パス)
その後、再度命令を1周し、バイナリーコードへ変換します。(第2パス)
実装面では、SymbolTableクラスを追加し、シンボルテーブルを作成します。
また、Hackクラスにdo_binary_conversionメソッドを追加し、第1パスでシンボルテーブルを作成し、第2パスでバイナリーコードへ変換します。
第1パスでは、ラベル宣言を格納するための行番号記録用を用意し、第2パスでは、変数シンボル参照をもつA命令について、シンボルテーブルでシンボルを検索し、なければ追加します。
```python
class SymbolTable:
    def __init__(self):
        self.table = {}
        self.table["R0"] = 0
        self.table["R1"] = 1
        self.table["R2"] = 2
        self.table["R3"] = 3
        self.table["R4"] = 4
        self.table["R5"] = 5
        self.table["R6"] = 6
        self.table["R7"] = 7
        self.table["R8"] = 8
        self.table["R9"] = 9
        self.table["R10"] = 10
        self.table["R11"] = 11
        self.table["R12"] = 12
        self.table["R13"] = 13
        self.table["R14"] = 14
        self.table["R15"] = 15
        self.table["SP"] = 0
        self.table["LCL"] = 1
        self.table["ARG"] = 2
        self.table["THIS"] = 3
        self.table["THAT"] = 4
        self.table["SCREEN"] = 16384
        self.table["KBD"] = 24576
        
    def addEntry(self, symbol: str, address: int):
        self.table[symbol] = address
    
    def contains(self, symbol: str) -> bool:
        return True if symbol in self.table else False
    
    def getAddress(self, symbol: str) -> int:
        return self.table[symbol]
```
```python
class Hack:
    def __init__(self, asm_file_name: str) -> None:
        self.asm_file_path = f"./{asm_file_name}"
        self.parser = Parser(asm_file_path=self.asm_file_path)
        self.symbol_table = SymbolTable()
        # 出力ファイルのProg.hackを作成する
        self.hack_file_path = f"./{asm_file_name.replace('asm', 'hack')}"
        with open(self.hack_file_path, "w") as fp:
            fp.write("")

    def do_binary_conversion(self) -> None:
        """
        第1パスでシンボルテーブルを作成
        第2パスでバイナリコードへ変換
        """
        self.create_symbol_table()
        self.convert_asm_to_hack()
        
    def create_symbol_table(self) -> None:
        """
        第1パスで用いられる。ここでは、すべてのラベルシンボルがテーブルへ追加される。
        変数シンボルについては、第2パスで追加される。
        """
        # ラベル宣言を格納するための行番号記録用
        line_number = 0
        tmp_parser = Parser(asm_file_path=self.asm_file_path)
        # tmp_parserを用いて、命令を一周し、シンボルテーブルを作成する
        while tmp_parser.has_more_lines():
            tmp_parser.advance()
            if tmp_parser.order is None:
                continue
            if tmp_parser.instruction_type() == Instruction.L:
                symbol = tmp_parser.symbol()
                self.symbol_table.addEntry(symbol=symbol, address=line_number)
                continue
            line_number += 1
        del tmp_parser

    def convert_asm_to_hack(self) -> None:
        # 各行(アセンブリ命令)を反復処理する
        # C命令については、各フィールドをバイナリーコードに変換して、連結する
        # A命令については、xxxをバイナリーコードに変換する
        # 変数シンボル参照をもつA命令については、シンボルテーブルでシンボルを検索し、なければ追加する。
        # 変数シンボルを格納するためのRAM番号記録用(val_number>=16)
        val_number = 16
        
        while self.parser.has_more_lines():
            self.parser.advance()
            if self.parser.order is None:
                continue

            binary_code = ""
            if self.parser.instruction_type() == Instruction.A:
                symbol = self.parser.symbol()
                # A命令のxxxが整数かどうかで変数シンボルかを判定する
                # symbol_is_intがFalseならば、変数シンボル
                symbol_is_int = symbol.isdigit()
                if symbol_is_int:
                    symbol_int = int(symbol)
                else:
                    # 変数シンボルの場合
                    if self.symbol_table.contains(symbol=symbol):
                        address = self.symbol_table.getAddress(symbol=symbol)
                        symbol_int = int(address)
                    else:
                        self.symbol_table.addEntry(symbol=symbol, address=val_number)
                        symbol_int = int(val_number)
                        val_number += 1
                # 0埋めの16bitバイナリへ変換
                binary_16bit_str = f"{symbol_int:016b}"
                binary_code += binary_16bit_str
            elif self.parser.instruction_type() == Instruction.C:
                dest_asm = self.parser.dest()
                comp_asm = self.parser.comp()
                jump_asm = self.parser.jump()
                
                dest_binary = Code.dest(dest_asm)
                comp_binary = Code.comp(comp_asm)
                jump_binary = Code.jump(jump_asm)
                
                binary_code += "111"
                is_a = "1" if "M" in comp_asm else "0"
                binary_code += is_a
                binary_code += comp_binary
                binary_code += dest_binary
                binary_code += jump_binary
            elif self.parser.instruction_type() == Instruction.L:
                # 何もせず書き込まない
                continue

            with open(self.hack_file_path, "a") as fp:
                fp.write(binary_code)
                fp.write("\n")
```

## まとめ
Nand2Tetrisの6章であるアセンブラをPythonを用いて実装しました。
この章では、1~5章の実施内容に加えて、Tetrisに向けて積み上げている感覚が得られました（楽しいいいい）
みなさまが6章を読まれる際に、少しでも参考になればうれしいです！
次章からソフトウェア編に入りますが、また別途記事にまとめます。