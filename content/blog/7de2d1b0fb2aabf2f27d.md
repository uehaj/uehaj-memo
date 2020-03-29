---
date: 2020-02-20T15:08:12.174Z
path: /7de2d1b0fb2aabf2f27d.md/index.html
template: BlogPost
title: 第28回 オフラインリアルタイムどう書くの問題「十字の壁がそそり立つ世界の中を君は螺旋状に歩く」をRustで解く
tags: Rust どう書く
author: uehaj
slide: false
---
[第28回 オフラインリアルタイムどう書くの問題「十字の壁がそそり立つ世界の中を君は螺旋状に歩く」](http://qiita.com/Nabetani/items/ce3b4f8a34de8987c6bc)を、Rust(2015-02-22 nightly)で解きました。

u32だったのをu64にして、末尾再帰をloopに置き換えただけで、追加問題も難なく実行できました。

# 感想

* このレベル(並列なし、Owned Pointerなし、traitなし)だと、Rustは単に「すばらしいC言語」。何もデメリットがない。C言語経験者だったら、単に楽しく便利なだけ。
  * Boxとか駆使してライフタイムがごにょり始めると、コンパイルエラーを乗り越えるための苦痛が予想される…
* Rustは関数型プログラミングと相性が悪いと思う。&mutとかとどう折り合いつけるの? 逆に&mutを避けたいなら、Rust使う意味あんの？

# コード

```rust
#![allow(unused_variables)]
#![allow(dead_code)]

static DIREC_STR:[&'static str;4] = ["E", "S", "W", "N"];
static TURN_RIGHT:i32 = 1;
static TURN_LEFT:i32 = -1;
static FORWARD:i32 = 0;

#[derive(Debug)]
struct Segment {
    steps: u64,
    next_direc: i32
}

impl Segment {
    fn new(steps: u64, next_direc: i32) -> Segment {
        Segment { next_direc:next_direc, steps:steps }
    }
}

#[derive(Debug)]
struct You {
    pos: u64,
    direc: u32
}

fn trace_segment_and_turn(day_of_answer:u64, segment: &Segment, you: &mut You)
{
    if day_of_answer < you.pos + (*segment).steps  {
        you.pos = day_of_answer;
        return;
    }
    you.pos += (*segment).steps;
    you.direc = ((you.direc as i32+(*segment).next_direc + 4) % 4) as u32;
}

fn create_segments(n:u64, e:u64, s:u64, w:u64, wall_thickness: u64) -> Vec<Segment>
{
    let mut segments = vec![
        Segment::new(e,              TURN_RIGHT),
        Segment::new(wall_thickness, TURN_RIGHT),
        Segment::new(e,              TURN_LEFT),
        Segment::new(s,              TURN_RIGHT),
        Segment::new(wall_thickness, TURN_RIGHT),
        Segment::new(s,              TURN_LEFT),
        Segment::new(w,              TURN_RIGHT),
        Segment::new(wall_thickness, TURN_RIGHT),
        Segment::new(w,              TURN_LEFT),
        Segment::new(n,              TURN_RIGHT)];
    if n > 1 {
        segments.push(Segment::new(wall_thickness, TURN_RIGHT));
        segments.push(Segment::new(n-1,   TURN_LEFT));
    } else {
        segments.push(Segment::new(wall_thickness, FORWARD));
    }
    segments
}

fn trace_path(day_of_answer:u64, you: &mut You,
              n:u64, e:u64, s:u64, w:u64, mut wall_thickness: u64) {
    loop {
        for segment in create_segments(n, e, s, w, wall_thickness).iter() {
            trace_segment_and_turn(day_of_answer, segment, you);
            if you.pos >= day_of_answer { return; }
        }
        you.pos += 1;
        wall_thickness += 2;
    }
}

fn solve(e:u64, s:u64, w:u64, n:u64, day_of_answer:u64) -> &'static str {
    let mut you = You {pos:0, direc:0};
    trace_path(day_of_answer, &mut you, e, s, w, n, 2);
    return DIREC_STR[you.direc as usize]
}

fn test(e:u64, s:u64, w:u64, n:u64, day_of_answer:u64, expected:&str) {
    assert_eq!(solve(e, s, w, n, day_of_answer), expected);
}

#[test]
fn test_case() {
    /*0*/ test( 2, 3, 5, 4, 85, "S" );
    /*1*/ test( 1, 2, 3, 4, 1, "E" );
    /*2*/ test( 1, 2, 3, 4, 2, "S" );
    /*3*/ test( 1, 2, 3, 4, 3, "S" );
    /*4*/ test( 1, 2, 3, 4, 4, "W" );
    test( 2, 3, 5, 4,  85, "S");
    test( 1, 2, 3, 4,  1, "E");
    test( 1, 2, 3, 4,  2, "S");
    test( 1, 2, 3, 4,  3, "S");
    test( 1, 2, 3, 4,  4, "W");
    test( 1, 2, 3, 4,  27, "E");
    test( 1, 2, 3, 4,  63, "E");
    test( 1, 2, 3, 4,  40, "W");
    test( 1, 4, 3, 2,  40, "S");
    test( 3, 3, 3, 3,  30, "S");
    test( 3, 3, 3, 3,  31, "E");
    test( 3, 3, 3, 3,  32, "E");
    test( 3, 3, 3, 3,  70, "S");
    test( 3, 3, 3, 3,  71, "E");
    test( 3, 3, 3, 3,  72, "E");
    test( 1, 1, 1, 1,  7, "N");
    test( 1, 2, 1, 1,  7, "W");
    test( 1, 6, 1, 1,  7, "S");
    test( 1, 8, 1, 1,  7, "E");
    test( 1, 1, 1, 1,  30, "N");
    test( 1, 2, 1, 1,  30, "W");
    test( 1, 5, 1, 1,  30, "S");
    test( 1, 8, 1, 1,  30, "E");
    test( 9, 9, 9, 9,  99, "W");
    test( 5, 6, 3, 8,  3, "E");
    test( 5, 8, 1, 1,  11, "W");
    test( 2, 8, 1, 2,  18, "S");
    test( 3, 2, 3, 1,  20, "N");
    test( 3, 3, 8, 1,  28, "N");
    test( 2, 5, 1, 2,  32, "E");
    test( 2, 5, 1, 6,  33, "E");
    test( 1, 2, 5, 7,  34, "N");
    test( 3, 6, 5, 6,  36, "E");
    test( 6, 2, 8, 1,  39, "S");
    test( 3, 1, 2, 3,  41, "W");
    test( 1, 1, 3, 4,  45, "W");
    test( 1, 3, 1, 2,  46, "N");
    test( 4, 4, 4, 4,  49, "W");
    test( 3, 1, 4, 4,  55, "N");
    test( 6, 6, 2, 1,  56, "W");
    test( 3, 2, 1, 2,  59, "S");
    test( 2, 7, 7, 1,  60, "S");
    test( 3, 1, 1, 1,  63, "N");
    test( 4, 6, 4, 1,  78, "E");
    test( 7, 5, 3, 6,  79, "W");
    test( 7, 8, 3, 1,  81, "E");
    test( 3, 2, 5, 2,  82, "S");
    test( 1, 1, 3, 4,  84, "N");
    test( 7, 4, 1, 5,  88, "S");
    test( 3, 6, 5, 3,  89, "S");
    test( 1, 4, 2, 3,  92, "N");
    test( 1, 3, 4, 5,  93, "W");
    test( 2, 4, 8, 1,  94, "W");
    test( 3, 6, 1, 7,  99, "S");
}

fn main() {
    assert_eq!(solve(1234, 2345, 3456, 4567, 978593417), "E");
    assert_eq!(solve(1234, 2345, 3456, 4567, 978593418), "S");
    assert_eq!(solve(31415, 92653, 58979, 32384, 9812336139), "W");
    assert_eq!(solve(31415, 92653, 58979, 32384, 9812336140), "S");
    assert_eq!(solve(314159, 265358, 979323, 84626, 89099331642), "S");
    assert_eq!(solve(314159, 265358, 979323, 84626, 89099331643), "W");
}
```


