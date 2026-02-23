# NESエミュレータ開発の壁（C-2：Rustで解剖するROMローダー）

前回（C-1）で、`.nes` ファイルの先頭16バイト（iNESヘッダー）に、ゲームカセットの構成データがすべて詰め込まれていることを学びました。

今回は、エミュレータを起動して一番最初に動く**「ROMファイルの読み込み（パース）関数」**をRustでどう実装するかを解説します。

---

## 4. `Rom` 構造体の設計

読み解いたデータを収納するための「カセット」を表現する構造体を作ります。

```rust
pub struct Rom {
    pub prg_rom: Vec<u8>,        // プログラムデータ本体
    pub chr_rom: Vec<u8>,        // グラフィックデータ本体
    pub mapper: u8,              // どの特殊チップを使うか(0〜255)
    pub screen_mirroring: Mirroring, // 水平か？垂直か？
}

#[derive(Debug, PartialEq)]
pub enum Mirroring {
    Vertical,
    Horizontal,
    FourScreen,
}
```

---

## 5. iNESヘッダーの解剖コード

生のバイトデータ（`raw: &Vec<u8>`）を引数として受け取り、先頭から順番に読み進めていくパース関数 `new()` を実装します。

```rust
// 定数：各サイズの基本ブロック（バイト数）
const PRG_ROM_PAGE_SIZE: usize = 16384; // 16KB
const CHR_ROM_PAGE_SIZE: usize = 8192;  // 8KB

impl Rom {
    pub fn new(raw: &Vec<u8>) -> Result<Rom, String> {
        // ---- 1. マジックナンバーの確認 (最初の4文字) ----
        if &raw[0..4] != b"NES\x1A" {
            return Err("これはファミコン(iNES)のRomファイルではありません！".to_string());
        }

        // ---- 2. マッパー番号とミラーリングの復元 (バイト6・7) ----
        // バイト6からミラーリング設定等を読み取る
        let control_1 = raw[6];
        let control_2 = raw[7];

        // 4画面ミラーリングか？ (bit3)
        let is_four_screen = control_1 & 0b1000 != 0;
        
        // 垂直(bit0=1) か 水平(bit0=0) か？
        let is_vertical_mirroring = control_1 & 0b0001 != 0;
        let screen_mirroring = match (is_four_screen, is_vertical_mirroring) {
            (true, _) => Mirroring::FourScreen,
            (false, true) => Mirroring::Vertical,
            (false, false) => Mirroring::Horizontal,
        };

        // マッパー番号の復元（過去の魔改造の結合）
        // バイト6の上位4ビットを右にシフトして下半分にし、
        // バイト7の上位4ビットはそのまま（左半分の位置）にして、足し合わせる！
        let mapper = (control_1 >> 4) | (control_2 & 0b1111_0000);

        // ---- 3. trainer（チート用のおまけデータ）のスキップ ----
        // trainer(容量512バイト)がある場合は、本体データ開始位置をズラす
        let has_trainer = control_1 & 0b0100 != 0;
        let prg_rom_start = 16 + if has_trainer { 512 } else { 0 };

        // ---- 4. PRG-ROM (プログラム) の抽出 ----
        let prg_rom_size = raw[4] as usize * PRG_ROM_PAGE_SIZE; // (例: 2 * 16384 = 32768)
        let prg_rom_end = prg_rom_start + prg_rom_size;
        
        // ベクタのスライスを使って、プログラム本体だけを綺麗に切り抜く！
        let prg_rom = raw[prg_rom_start..prg_rom_end].to_vec();

        // ---- 5. CHR-ROM (グラフィック) の抽出 ----
        let chr_rom_start = prg_rom_end; // CHRはPRGの直後から始まる
        let chr_rom_size = raw[5] as usize * CHR_ROM_PAGE_SIZE; // (例: 1 * 8192)
        
        let chr_rom = if chr_rom_size == 0 {
            // 例外：カセットにCHRが乗ってなく、本体のRAMを使うゲーム(CHR-RAM)の場合は空箱を作る
            vec![0; CHR_ROM_PAGE_SIZE]
        } else {
            raw[chr_rom_start..(chr_rom_start + chr_rom_size)].to_vec()
        };

        // ---- 解析完了！無事に構造体に詰めて返す ----
        Ok(Rom {
            prg_rom: prg_rom,
            chr_rom: chr_rom,
            mapper: mapper,
            screen_mirroring: screen_mirroring,
        })
    }
}
```

### ビット演算の意味の確認

途中で出てきたマッパー結合のビット演算が少し難しく見えるかもしれません。

`let mapper = (control_1 >> 4) | (control_2 & 0b1111_0000);`

*   `control_1`（バイト6）の `1010_0110` のようなデータから、上の `1010` をマッパー番号として使いたいです。
*   `>> 4` と右に4回ずらすことで、`0000_1010` にします。これがマッパーの「下1桁」になります。
*   `control_2`（バイト7）の `0011_0000` などのデータから、同じく上の `0011` を使いたいです。
*   しかしこれは上半分がそのままマッパーの「上1桁」になるので、`& 0b1111_0000` で下位4つを綺麗に削ぎ落として `0011_0000` にします。
*   最後にこの2つを OR（`|` : 足し算） することで、`0011_1010` （16進数で `0x3A`＝10進数でマッパー58番！）という1つの綺麗な番号が浮かび上がるのです。

---

## 6. すべての冒険の入り口

この美しく整備された `Rom` クラスが完成すれば、エミュレータの `main.rs` 関数は、以下のようにゲームファイルを起動することができるようになります。

```rust
// ファイルをそのまま読み込み、バイトの配列(Vec<u8>)にする
let raw_file = std::fs::read("super_mario.nes").expect("ファイルがありません！");

// 先ほどの魔法のパース関数に投げる
let cartridge = Rom::new(&raw_file).expect("Romパース失敗！");

// 「あ！マッパー0番で、垂直ミラーリングだ！」という情報を取り出して、
// エミュレータのバス（Bus）にガチャンと挿し込む
let mut bus = Bus::new(cartridge); 
let mut cpu = Cpu::new(bus);

// 冒険（エミュレーションループ）をスタート！
cpu.reset();
cpu.run();
```

このように、ブラックボックスのように見えたバイナリファイル（`.nes`）も、最初の16文字という**「先人たちが取り決めたルールの仕様書」**さえ読めば、いくらでも中身を切り刻んで自分の好きな形に整理することが可能なのです。

エミュレータ開発とは、ただプログラミング言語を書くことではなく、こうした「遠い昔のハードウェアの制約と、ソフトウェアの魔改造の歴史」を紐解く、古文書の解読のような作業でもあるのです。
