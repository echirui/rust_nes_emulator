# 別紙：6502 CPU オペコード・マトリックス

エミュレータの実装で欠かせない、CPU（6502）の「16進数オペコード」「アドレッシングモード」「バイト数」「消費クロック数」の対応表です。

## 📜 サイクル数の追加ルール（+1の意味）
* **`+p` (ページクロス)**: インデックス修飾（XやYを足す）によってアドレスの下位8ビットが繰り上がり、上位8ビットが変化した場合、メモリアクセスのため**さらに1サイクル**消費します。
* **`+b` (ブランチ)**: `BEQ`などの分岐命令で、条件が成立して実際に飛んだ場合に**+1サイクル**。さらに、飛んだ先が現在と違うページだった場合は**もう+1サイクル**消費します。

<br>

## 🔢 16x16 オペコード・マトリックス (早見表)

左の列（下位1桁）と上の行（上位1桁）を組み合わせて調べます。
*(空欄は非公式(Undocumented)命令または未定義です)*

| 下位 \ 上位 | 0x0_ | 0x1_ | 0x2_ | 0x3_ | 0x4_ | 0x5_ | 0x6_ | 0x7_ | 0x8_ | 0x9_ | 0xA_ | 0xB_ | 0xC_ | 0xD_ | 0xE_ | 0xF_ |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **_0** | BRK | BPL | JSR | BMI | RTI | BVC | RTS | BVS | | BCC | LDY | BCS | CPY | BNE | CPX | BEQ |
| **_1** | ORA | ORA | AND | AND | EOR | EOR | ADC | ADC | STA | STA | LDA | LDA | CMP | CMP | SBC | SBC |
| **_2** | | | | | | | | | | | LDX | | | | | |
| **_3** | | | | | | | | | | | | | | | | |
| **_4** | | | BIT | | | | | | STY | STY | LDY | LDY | CPY | | CPX | |
| **_5** | ORA | ORA | AND | AND | EOR | EOR | ADC | ADC | STA | STA | LDA | LDA | CMP | CMP | SBC | SBC |
| **_6** | ASL | ASL | ROL | ROL | LSR | LSR | ROR | ROR | STX | STX | LDX | LDX | DEC | DEC | INC | INC |
| **_7** | | | | | | | | | | | | | | | | |
| **_8** | PHP | CLC | PLP | SEC | PHA | CLI | PLA | SEI | DEY | TYA | TAY | CLV | INY | CLD | INX | SED |
| **_9** | ORA | ORA | AND | AND | EOR | EOR | ADC | ADC | | STA | LDA | LDA | CMP | CMP | SBC | SBC |
| **_A** | ASL | | ROL | | LSR | | ROR | | TXA | TXS | TAX | TSX | DEX | | NOP | |
| **_B** | | | | | | | | | | | | | | | | |
| **_C** | | | BIT | | JMP | | JMP | | STY | | LDY | LDY | CPY | | CPX | |
| **_D** | ORA | ORA | AND | AND | EOR | EOR | ADC | ADC | STA | STA | LDA | LDA | CMP | CMP | SBC | SBC |
| **_E** | ASL | ASL | ROL | ROL | LSR | LSR | ROR | ROR | STX | | LDX | LDX | DEC | DEC | INC | INC |
| **_F** | | | | | | | | | | | | | | | | |

<br>

## 📚 命令別 詳細リスト（バイト数・サイクル数）

### 📥 ロード・ストア転送系 (Load / Store / Transfer)
| 命令 | 概要 | モード | Op | バイト | サイクル |   | 命令 | 概要 | モード | Op | バイト | サイクル |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **LDA** | Aへロード | Imm | 0xA9 | 2 | 2 | | **STA** | Aを保存 | ZP | 0x85 | 2 | 3 |
| | | ZP | 0xA5 | 2 | 3 | | | | ZP,X | 0x95 | 2 | 4 |
| | | ZP,X | 0xB5 | 2 | 4 | | | | Abs | 0x8D | 3 | 4 |
| | | Abs | 0xAD | 3 | 4 | | | | Abs,X | 0x9D | 3 | 5 |
| | | Abs,X | 0xBD | 3 | 4+p | | | | Abs,Y | 0x99 | 3 | 5 |
| | | Abs,Y | 0xB9 | 3 | 4+p | | | | Ind,X | 0x81 | 2 | 6 |
| | | Ind,X | 0xA1 | 2 | 6 | | | | Ind,Y | 0x91 | 2 | 6 |
| | | Ind,Y | 0xB1 | 2 | 5+p | | **STX** | Xを保存 | ZP | 0x86 | 2 | 3 |
| **LDX** | Xへロード | Imm | 0xA2 | 2 | 2 | | | | ZP,Y | 0x96 | 2 | 4 |
| | | ZP | 0xA6 | 2 | 3 | | | | Abs | 0x8E | 3 | 4 |
| | | ZP,Y | 0xB6 | 2 | 4 | | **STY** | Yを保存 | ZP | 0x84 | 2 | 3 |
| | | Abs | 0xAE | 3 | 4 | | | | ZP,X | 0x94 | 2 | 4 |
| | | Abs,Y | 0xBE | 3 | 4+p | | | | Abs | 0x8C | 3 | 4 |
| **LDY** | Yへロード | Imm | 0xA0 | 2 | 2 | | **TAX** | A->X転送 | Imp | 0xAA | 1 | 2 |
| | | ZP | 0xA4 | 2 | 3 | | **TAY** | A->Y転送 | Imp | 0xA8 | 1 | 2 |
| | | ZP,X | 0xB4 | 2 | 4 | | **TXA** | X->A転送 | Imp | 0x8A | 1 | 2 |
| | | Abs | 0xAC | 3 | 4 | | **TYA** | Y->A転送 | Imp | 0x98 | 1 | 2 |
| | | Abs,X | 0xBC | 3 | 4+p | | **TSX** | SP->X | Imp | 0xBA | 1 | 2 |
| | | | | | | | **TXS** | X->SP | Imp | 0x9A | 1 | 2 |

### 🧮 算術・論理演算 (Arithmetic / Logic)
| 命令 | 概要 | モード | Op | バイト | サイクル |   | 命令 | 概要 | モード | Op | バイト | サイクル |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **ADC** | キャリー付加算| Imm | 0x69 | 2 | 2 | | **SBC** | キャリー付減算| Imm | 0xE9 | 2 | 2 |
| | | ZP | 0x65 | 2 | 3 | | | | ZP | 0xE5 | 2 | 3 |
| | | ZP,X | 0x75 | 2 | 4 | | | | ZP,X | 0xF5 | 2 | 4 |
| | | Abs | 0x6D | 3 | 4 | | | | Abs | 0xED | 3 | 4 |
| | | Abs,X | 0x7D | 3 | 4+p | | | | Abs,X | 0xFD | 3 | 4+p |
| | | Abs,Y | 0x79 | 3 | 4+p | | | | Abs,Y | 0xF9 | 3 | 4+p |
| | | Ind,X | 0x61 | 2 | 6 | | | | Ind,X | 0xE1 | 2 | 6 |
| | | Ind,Y | 0x71 | 2 | 5+p | | | | Ind,Y | 0xF1 | 2 | 5+p |
| **AND** | 論理積(AND) | (ADCと同一) | ... | .. | . | | **CMP** | Aとメモリ比較 | (ADCと同一) | ... | .. | . |
| **ORA** | 論理和(OR) | (ADCと同一) | ... | .. | . | | **CPX** | Xと比較 | Imm | 0xE0 | 2 | 2 |
| **EOR** | 排他的論理和 | (ADCと同一) | ... | .. | . | | | | ZP | 0xE4 | 2 | 3 |
| **BIT** | ビットテスト | ZP | 0x24 | 2 | 3 | | | | Abs | 0xEC | 3 | 4 |
| | | Abs | 0x2C | 3 | 4 | | **CPY** | Yと比較 | Imm | 0xC0 | 2 | 2 |
| | | | | | | | | | ZP | 0xC4 | 2 | 3 |
| | | | | | | | | | Abs | 0xCC | 3 | 4 |

### 🔄 インクリメント・シフト系 (Inc / Dec / Shift)
| 命令 | 概要 | モード | Op | バイト | サイクル |   | 命令 | 概要 | モード | Op | バイト | サイクル |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **INC** | メモリ+1 | ZP | 0xE6 | 2 | 5 | | **ASL** | 左シフト(1bit)| Acc | 0x0A | 1 | 2 |
| | | ZP,X | 0xF6 | 2 | 6 | | | | ZP | 0x06 | 2 | 5 |
| | | Abs | 0xEE | 3 | 6 | | | | ZP,X | 0x16 | 2 | 6 |
| | | Abs,X | 0xFE | 3 | 7 | | | | Abs | 0x0E | 3 | 6 |
| **DEC** | メモリ-1 | ZP | 0xC6 | 2 | 5 | | | | Abs,X | 0x1E | 3 | 7 |
| | | ZP,X | 0xD6 | 2 | 6 | | **LSR** | 右シフト(1bit)| (ASLと同一) |... | .. | . |
| | | Abs | 0xCE | 3 | 6 | | **ROL** | 左ローテート | (ASLと同一) |... | .. | . |
| | | Abs,X | 0xDE | 3 | 7 | | **ROR** | 右ローテート | (ASLと同一) |... | .. | . |
| **INX** | Xを+1 | Imp | 0xE8 | 1 | 2 | | | | | | | |
| **INY** | Yを+1 | Imp | 0xC8 | 1 | 2 | | | | | | | |
| **DEX** | Xを-1 | Imp | 0xCA | 1 | 2 | | | | | | | |
| **DEY** | Yを-1 | Imp | 0x88 | 1 | 2 | | | | | | | |

### 🔀 分岐・スタック・フラグ系 (Branch / Jump / Stack)
| 命令 | 概要 | モード | Op | バイト | サイクル |   | 命令 | 概要 | モード | Op | バイト | サイクル |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **JMP** | ジャンプ | Abs | 0x4C | 3 | 3 | | **PHA** | AをPush | Imp | 0x48 | 1 | 3 |
| | | Ind | 0x6C | 3 | 5 | | **PLA** | AをPop | Imp | 0x68 | 1 | 4 |
| **JSR** | 関数へ飛ぶ | Abs | 0x20 | 3 | 6 | | **PHP** | PをPush | Imp | 0x08 | 1 | 3 |
| **RTS** | 関数から戻る | Imp | 0x60 | 1 | 6 | | **PLP** | PをPop | Imp | 0x28 | 1 | 4 |
| **RTI** | 割込から戻る | Imp | 0x40 | 1 | 6 | | | | | | | |
| **BCC** | Cary=0で分岐 | Rel | 0x90 | 2 | 2+b | | **CLC** | Caryをクリア | Imp | 0x18 | 1 | 2 |
| **BCS** | Cary=1で分岐 | Rel | 0xB0 | 2 | 2+b | | **SEC** | Caryをセット | Imp | 0x38 | 1 | 2 |
| **BNE** | Zero=0で分岐 | Rel | 0xD0 | 2 | 2+b | | **CLI** | Intlをクリア | Imp | 0x58 | 1 | 2 |
| **BEQ** | Zero=1で分岐 | Rel | 0xF0 | 2 | 2+b | | **SEI** | Intlをセット | Imp | 0x78 | 1 | 2 |
| **BPL** | Nega=0で分岐 | Rel | 0x10 | 2 | 2+b | | **CLV** | OverFをクリア | Imp | 0xB8 | 1 | 2 |
| **BMI** | Nega=1で分岐 | Rel | 0x30 | 2 | 2+b | | **CLD** | Deciをクリア | Imp | 0xD8 | 1 | 2 |
| **BVC** | OverF=0分岐| Rel | 0x50 | 2 | 2+b | | **SED** | Deciをセット | Imp | 0xF8 | 1 | 2 |
| **BVS** | OverF=1分岐| Rel | 0x70 | 2 | 2+b | | | | | | | |
