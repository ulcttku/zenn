---
title: "CamelCase な型と SnakeCase な型の変換"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript"]
published: true
---

TypeScript で、(Lower)CamelCase な型を SnakeCase な型に、SnakeCase な型を(Lower)CamelCase な型に変換する型を作りました。
具体的には、

```ts
type camelCase = SnakeToCamel<"camel_case">;
// type camelCase = "camelCase"

type snakeCase = CamelToSnake<"snakeCase">;
// type snakeCase = "snake_case";
```
という感じです。

とくに (Lower)CamelCase から SnakeCase に変換する型を作るときに苦労したので、軽く解説を残しておこうかと思います。

TypeScript のバージョンは 4.3.4 です。

# TL;DR

作った型だけ見たい人のために。

## SnakeCase → CamelCase

```ts
 type SnakeToCamel<T extends string> =
    T extends `${infer Head}_${infer Tail}` ?
        `${Head}${SnakeToCamel<Capitalize<Tail>>}` :
        T;
```

## CamelCase → SnakeCase

```ts
type CamelToSnakeCap<T extends string, Separator extends string> = 
    T extends `${infer Head}${Separator}${infer Tail}` ?
        `${CamelToSnakeCap<Head, Separator>}_${CamelToSnakeCap<`${Uncapitalize<Separator>}${Tail}`, Separator>}` :
        T;

type CamelAToSnakeMany<T extends string, Separators extends string> =
    Separators extends `${infer Separator}${infer RestSeparators}` ?
        RestSeparators extends "" ? 
            CamelToSnakeCap<T, Extract<Separator, string>> :
            CamelAToSnakeMany<CamelToSnakeCap<T, Extract<Separator, string>>, Extract<RestSeparators, string>> :
        "";

type CamelToSnake<T extends string> =
    CamelAToSnakeMany<CamelAToSnakeMany<T, "ABCDEFGHIJKLM">,"NOPQRSTUVWXYZ">;
```


# 宣伝?

ここで紹介する型は aikagi という npm ライブラリを作る際に作ったものです。
サーバーから送られてくる SnakeCase のキーをもつ Object を CamelCase のキーをもつ Object に型を維持したまま変換するライブラリとなっています。

もしよければ触ってみてもらえると嬉しいです。

https://www.npmjs.com/package/aikagi


# SnakeCase → CamelCase

型としては以下の通りです。
```ts
 type SnakeToCamel<T extends string> =
    T extends `${infer Head}_${infer Tail}` ?
        `${Head}${SnakeToCamel<Capitalize<Tail>>}` :
        T;
```

思いついた処理をそのまま実装したような形なので、あまり解説するようなことはないかと思いますが、処理としては

1. string に割り当て可能な型 (`T`) を受け取る
2. `_` の前 (`Head`) と後ろ (`Tail`) で分割
3. 分割できれば
   1. `Tail` の1文字目を大文字に変換 (`Tail'` とする)
   2. `Head` と、`Tail'` を 1. から処理したものを連結して返す
4. 分割できなければ
   1. そのまま受け取った型 (`T`) を返す

という感じです。

※この実装には少し問題点があるのですが、後ほど取り上げます。


# CamelCase → SnakeCase

いろいろ試行錯誤したので、試した方針を1つひとつ解説していこうと思います。

## 方針1

SnakeCase から CamelCase への変換と同じように、目印として大文字を使おうかと思いましたが、大文字は26種類もあって同じよう実装できなさそうなので諦めます。
とりあえず、先頭から順に見ていって大文字があれば `_` を追加して小文字にする、という方針を採りました。

作成した型はこちらです。
```ts
type CamelToSnake1<T extends string> =
    T extends `${infer Head}${infer Tail}` ?
        Head extends Capitalize<Head> ?
            `_${Lowercase<Head>}${CamelToSnake1<Tail>}` :
            `${Head}${CamelToSnake1<Tail>}` :
        T;
```

今回、想定しているのは LowerCamelCase なので先頭が大文字の場合は無視しています。

この処理の肝となる大文字の判定ですが、その判定をしているのは
```ts
Head extends Capitalize<Head> ? 真 : 偽
```
の部分です。

`Capitalize` は文字列の最初の文字を大文字に変換してくれる型です。

なので、`Head` が小文字、たとえば `"a"` の場合、
```ts
"a" extends Capitalize<"a"> ? 真 : 偽
```
は
```ts
"a" extends "A" ? 真 : 偽
```
となって偽が返ってきます。

また、`Head` が大文字、たとえば `"A"` の場合、
```ts
"A" extends Capitalize<"A"> ? 真 : 偽
```
は
```ts
"A" extends "A" ? 真 : 偽
```
となって真が返ってきます。


この実装はシンプルに書けてよかったのですが、実際に使ってみると文字数制限が厳しすぎるという問題が発生しました。

たとえば、短い文字列では
```ts
type success1 = CamelToSnake1<"isAdmin">;
// type success1 = "is_admin"
type success2 = CamelToSnake1<"latestComments">;
// type success2 = "latest_comments"
```
のように問題なく変換できていたのですが、
```ts
type failure1 = CamelToSnake1<"isOrganizationAdmin">;
// Type instantiation is excessively deep and possibly infinite.ts(2589)
type failure2 = CamelToSnake1<"relatedArticleId">;
// Type instantiation is excessively deep and possibly infinite.ts(2589)
```
という具合で、16文字を超えるとエラーとなります。

これは型の再帰が深すぎるというエラーで、[TypeScriptの型で遊ぶ時、再帰制限を（合法的に）突破する - Qiita](https://qiita.com/kazatsuyu/items/44c1b012d66aae1dc2c0) によると、再帰を50回するとエラーとなるようです。

TypeScript 本体のコードを読み解けなかったので、いろいろ試してみたところ Conditional Types の回数と Template Literal Types のテンプレート展開の合計が47回を超えるとエラーとなるようでした。(要検証)

`CamelToSnake1` の処理が進むと Conditional Types が2回、テンプレート展開が1回行われるので合計3回です。
なので、15文字の場合は、
```
15 * 3 = 45 <= 47
```
となって制限に引っかからなさそうですが、17文字の場合は、
```
16 * 3 = 48 > 47
```
となって制限に引っかかりそうです。


実は SnakeCase → CamelCase の変換でもこのエラーが出てしまうのですが、文字数の制限ではなく `_` の数の制限になり、無理な使い方をされない限りは問題がないと思うので上記の実装のままとしています。
```ts
type success1 = SnakeToCamel<"a_b_c_d_e_f_g_h_i_j_k_l_m_n_o_p_q_r_s_t_u_v_w_x">;
// type success1 = "aBCDEFGHIJKLMNOPQRSTUVWX"
type success2 = SnakeToCamel<"abc_def_ghi_jkl_mno_pqr_stu_vwx_yza_bcd_efg_hij_klm_nop_qrs_tuv">;
// type success2 = "abcDefGhiJklMnoPqrStuVwxYzaBcdEfgHijKlmNopQrsTuv"

type failure = SnakeToCamel<"a_b_c_d_e_f_g_h_i_j_k_l_m_n_o_p_q_r_s_t_u_v_w_x_y">;
// Type instantiation is excessively deep and possibly infinite.ts(2589)
```


## 方針2

SnakeCase → CamelCase の変換のように大文字を目印として使う方針でいこうと思いましたが、合併型を使うと型推論がうまくいことがあるためダメでした。
```ts
type Separators = "A" | "B" | "C" | "D" | "E" | "F" | "G" | "H" | "I" | "J"
                | "K" | "L" | "M" | "N" | "O" | "P" | "Q" | "R" | "S" | "T"
                | "U" | "V" | "W" | "X" | "Y" | "Z";
type RemoveHead<String extends string, Head extends string> = String extends `${Head}${infer Tail}` ? Tail : String;

type CamelToSnake2<T extends string> =
    T extends `${infer Head}${Separators}${infer _}` ?
        `${Head}_${CamelToSnake2<Uncapitalize<RemoveHead<T, Head>>>}` :
        T;

type failure = CamelToSnake2<"isOrganizationAdmin">;
// type failure = "isOrganization_admin" | "isOrganization_organization_admin" | "is_admin" | "is_organization_admin"
```


## 方針3

方針2は合併型を使って失敗したので、SnakeCase → CamelCase のときと同じような方法で大文字1つひとつを変換していく方針を考えてみました。

たとえば `"A"` の場合、
```ts
type CamelToSnakeA<T extends string> = 
    T extends `${infer Head}A${infer Tail}` ?
        `${CamelToSnakeA<Head>}_${CamelToSnakeA<`a${Tail}`>}` :
        T;
```
とします。

この実装であれば、文字数の制限はなくなります。
```ts
type success1 = CamelToSnakeA<"isOrganizationAdmin">;
// type success1 = "isOrganization_admin"

type success2 = CamelToSnakeA<"bbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbbAbbb">
// type success2 = "bbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb_abbb"
```


次に行く前に `CamelToSnakeA` を `"A"` 以外でも使えるように変更します。
```ts
type CamelToSnakeCap<T extends string, Separator extends string> = 
    T extends `${infer Head}${Separator}${infer Tail}` ?
        `${CamelToSnakeCap<Head, Separator>}_${CamelToSnakeCap<`${Uncapitalize<Separator>}${Tail}`, Separator>}` :
        T;

type success1 = CamelToSnakeCap<"isOrganizationAdmin", "A">;
// type success1 = "isOrganization_admin"
type success2 = CamelToSnakeCap<"isOrganizationAdmin", "O">;
// type success2 = "is_organizationAdmin"
```

続いて大文字を順番に渡す処理を作っていきますが、型では `map` などは使えないので再帰を使います。
```ts
type CamelAToSnakeMany<T extends string, Separators extends string[]> =
    Separators extends [infer Separator, ...infer RestSeparators] ?
        RestSeparators extends [] ? 
            CamelToSnakeCap<T, Extract<Separator, string>> :
            CamelAToSnakeMany<CamelToSnakeCap<T, Extract<Separator, string>>, Extract<RestSeparators, string[]>> :
        "";

type success = CamelAToSnakeMany<"isOrganizationAdmin", ["A", "O"]>;
// type success = "is_organization_admin"
```
良さそうです。

それではすべてのアルファベットを渡してみます。

```ts
type CamelToSnake3_1<T extends string> =
    CamelAToSnakeMany<
        T,
        [
            "A", "B", "C", "D", "E", "F", "G", "H", "I", "J",
            "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T",
            "U", "V", "W", "X", "Y", "Z"
        ]
    >;
// Type instantiation is excessively deep and possibly infinite.ts(2589)
```

またこのエラーです…。
おそらく、渡している配列の再帰だけで50回制限を超えてしまったのかと思います。

ここで [TypeScriptの型で遊ぶ時、再帰制限を（合法的に）突破する - Qiita](https://qiita.com/kazatsuyu/items/44c1b012d66aae1dc2c0) の

> 天井が低いのでこれ以上皿が積めないなら、横にもう一個山を作ってしまえば良いのです。

というアイデアをお借りしようと思います。
ということで、分割してみたものがこちらです。
```ts
type CamelToSnake3_2<T extends string> =
    CamelAToSnakeMany<
        CamelAToSnakeMany<
            T,
            ["A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M"]
        >,
        ["N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "X", "Y", "Z"]
    >;
```

…はい、ただ分割しただけですが、エラーは出なくなりました。
それでは動作確認をしてみます。
```ts

type success = CamelToSnake3_2<"aBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZa">;
// type success = "a_bc_de_fg_hi_jk_lm_no_pq_rs_tu_vw_xy_za_bc_de_fg_hi_jk_lm_no_pq_rs_tu_vw_xy_za_bc_de_fg_hi_jk_lm_no_pq_rs_tu_vw_xy_za_bc_de_fg_hi_jk_lm_no_pq_rs_tu_vw_xy_za_bc_de_fg_hi_jk_lm_no_pq_rs_tu_vw_xy_za_bc_de_fg_hi_jk_lm_no_pq_rs_tu_vw_xy_za_bc_de_fg_hi_jk_lm_no_pq_rs_tu_vw_xy_za_bc_de_fg_hi_jk_lm_no_pq_rs_tu_vw_xy_za_bc_de_fg_hi_jk_lm_no_pq_rs_tu_vw_xy_za_bc_de_fg_hi_jk_lm_no_pq_rs_tu_vw_xy_za_bc_de_fg_hi_jk_lm_no_pq_rs_tu_vw_xy_za"
```
おぉ…思ったより長くまで大丈夫になりました。
これなら無理な使われ方をしても大丈夫そうです。

最後に、`CamelToSnake3_2` の見た目(長い配列)が嫌なので整理しようと思います。
修正すべきは `CamelAToSnakeMany` なので、もう一度載せておきます。

```ts
type CamelAToSnakeMany<T extends string, Separators extends string[]> =
    Separators extends [infer Separator, ...infer RestSeparators] ?
        RestSeparators extends [] ? 
            CamelToSnakeCap<T, Extract<Separator, string>> :
            CamelAToSnakeMany<CamelToSnakeCap<T, Extract<Separator, string>>, Extract<RestSeparators, string[]>> :
        "";
```

最初のループというイメージから `Separators` に配列を渡してしまっていますが、やっていることは先頭の1つとそれ以外への分割なので、文字列でも問題なさそうです。
なので、
```ts
type CamelAToSnakeMany<T extends string, Separators extends string> =
    Separators extends `${infer Separator}${infer RestSeparators}` ?
        RestSeparators extends "" ? 
            CamelToSnakeCap<T, Extract<Separator, string>> :
            CamelAToSnakeMany<CamelToSnakeCap<T, Extract<Separator, string>>, Extract<RestSeparators, string>> :
        "";
```
と書き直せます。

すると `CamelToSnake3_2` は
```ts
type CamelToSnake3_2<T extends string> =
    CamelAToSnakeMany<CamelAToSnakeMany<T, "ABCDEFGHIJKLM">,"NOPQRSTUVWXYZ">;
```
となり、見た目は綺麗になった気がします。

まとめると、
```ts
type CamelToSnakeCap<T extends string, Separator extends string> = 
    T extends `${infer Head}${Separator}${infer Tail}` ?
        `${CamelToSnakeCap<Head, Separator>}_${CamelToSnakeCap<`${Uncapitalize<Separator>}${Tail}`, Separator>}` :
        T;

type CamelAToSnakeMany<T extends string, Separators extends string> =
    Separators extends `${infer Separator}${infer RestSeparators}` ?
        RestSeparators extends "" ? 
            CamelToSnakeCap<T, Extract<Separator, string>> :
            CamelAToSnakeMany<CamelToSnakeCap<T, Extract<Separator, string>>, Extract<RestSeparators, string>> :
        "";

type CamelToSnake<T extends string> =
    CamelAToSnakeMany<CamelAToSnakeMany<T, "ABCDEFGHIJKLM">,"NOPQRSTUVWXYZ">;
```
となります。

# 感想

`infer` や `Template Literal Types` を使ったことがなかったのでいい勉強になりました。
それと再帰の制限とその対策は、なかなかトリッキーな感じがして面白かったです。

また、CamelCase → SnakeCase への変換はあまり綺麗な書き方ができなかったので、もっといい書き方があれば教えてもらえると嬉しいです。