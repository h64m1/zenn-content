---
title: "Zigで日付を取得する方法"
emoji: "📅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [zig, ziglang, date]
published: true
---

## std/time/epoch.zig

カレンダー日付に対応するタイムスタンプが必要な場合、`std.time.timestamp`関数（単位は秒）を使います。

https://ziglang.org/documentation/master/std/#root;time

実際のカレンダー日付（2022-12-17のような年月日の数値）を取得したい場合、`time`下にある`EpochSeconds`が使えます。`EpocSeconds`は`secs`プロパティとして、`timestamp`を受け取ります。

:::message
調べてみると、epochは0.9.0から実装された、割と新しい機能のようです。
https://ziglang.org/download/0.9.0/release-notes.html
:::

`EpocSeconds`が持つ関数や、その戻り値の`struct`を使うと、以下のようにして、年月日と時刻を取得できます。時刻はUTCベースのようなので、JSTに変換したい場合は手動でやるしかなさそう。

```zig
const std = @import("std");

fn testEpoch(secs: u64, expected: struct {
    year: u16,
    month: u4,
    day: u5,
    hours: u5,
    minutes: u6,
    seconds: u6,
}) !void {
    const epoch = std.time.epoch.EpochSeconds{ .secs = secs };
    const epoch_day = epoch.getEpochDay();
    const year_day = epoch_day.calculateYearDay();
    const month_day = year_day.calculateMonthDay();
    const day_seconds = epoch.getDaySeconds();

    const year = year_day.year;
    const month = month_day.month.numeric();
    const day = month_day.day_index + 1;
    const hours = day_seconds.getHoursIntoDay();
    const minutes = day_seconds.getMinutesIntoHour();
    const seconds = day_seconds.getSecondsIntoMinute();

    try std.testing.expectEqual(expected.year, year);
    try std.testing.expectEqual(expected.month, month);
    try std.testing.expectEqual(expected.day, day);
    try std.testing.expectEqual(expected.hours, hours);
    try std.testing.expectEqual(expected.minutes, minutes);
    try std.testing.expectEqual(expected.seconds, seconds);
}

test "EpochSeconds" {
    // 1970-01-01
    try testEpoch(0, .{
        .year = 1970,
        .month = 1,
        .day = 1,
        .hours = 0,
        .minutes = 0,
        .seconds = 0,
    });

    // 2022-12-18
    try testEpoch(1671330128, .{
        .year = 2022,
        .month = 12,
        .day = 18,
        .hours = 2,
        .minutes = 22,
        .seconds = 8,
    });
}
```

## 文字列に変換

取得された整数値を文字列に変換するには`format`を使います（`sprintf`的なやつ）。今回は試しに`yyyy-MM-dd`形式で出力してみました。以下のサイトに記載のように、`allocPrint`で`struct`に実装した`format`関数を呼び出す方式で出来ました。

参考：https://ziglearn.org/chapter-2/#formatting

```zig
const std = @import("std");

const Date = struct {
    year: u16,
    month: u4,
    day: u5,
    hours: u5,
    minutes: u6,
    seconds: u6,

    pub fn format(self: Date, comptime fmt: []const u8, options: std.fmt.FormatOptions, writer: anytype) !void {
        _ = fmt;
        _ = options;

        try writer.print("{d}", .{self.year});
        try writer.print("-{d:0>2}", .{self.month});
        try writer.print("-{d:0>2}", .{self.day});
    }
};

fn getDate(secs: u64) Date {
    const epoch = std.time.epoch.EpochSeconds{ .secs = secs };
    const epoch_day = epoch.getEpochDay();
    const year_day = epoch_day.calculateYearDay();
    const month_day = year_day.calculateMonthDay();
    const day_seconds = epoch.getDaySeconds();

    const year = year_day.year;
    const month = month_day.month.numeric();
    const day = month_day.day_index + 1;
    const hours = day_seconds.getHoursIntoDay();
    const minutes = day_seconds.getMinutesIntoHour();
    const seconds = day_seconds.getSecondsIntoMinute();

    return Date{
        .year = year,
        .month = month,
        .day = day,
        .hours = hours,
        .minutes = minutes,
        .seconds = seconds,
    };
}

fn testEpoch(secs: u64, expected: struct {
    year: u16,
    month: u4,
    day: u5,
    hours: u5,
    minutes: u6,
    seconds: u6,
    date: []const u8,
}) !void {
    const date = getDate(secs);

    try std.testing.expectEqual(expected.year, date.year);
    try std.testing.expectEqual(expected.month, date.month);
    try std.testing.expectEqual(expected.day, date.day);
    try std.testing.expectEqual(expected.hours, date.hours);
    try std.testing.expectEqual(expected.minutes, date.minutes);
    try std.testing.expectEqual(expected.seconds, date.seconds);

    const date_string = try std.fmt.allocPrint(std.testing.allocator, "{s}", .{date});
    defer std.testing.allocator.free(date_string);

    try std.testing.expectEqualStrings(expected.date, date_string);
}

test "EpochSeconds" {
    // 1970-01-01
    try testEpoch(0, .{
        .year = 1970,
        .month = 1,
        .day = 1,
        .hours = 0,
        .minutes = 0,
        .seconds = 0,
        .date = "1970-01-01",
    });

    // 2022-12-18
    try testEpoch(1671330128, .{
        .year = 2022,
        .month = 12,
        .day = 18,
        .hours = 2,
        .minutes = 22,
        .seconds = 8,
        .date = "2022-12-18",
    });
}
```

`format`の書式は、例えば以下に記載があって、
https://ziglang.org/documentation/master/std/#root;fmt.format

> The format string must be comptime-known and may contain placeholders following this format: {[argument][specifier]:[fill][alignment][width].[precision]}
> ...

今回の月と日のように、必ず二桁表示にして必要な場合に二桁目を0 paddingする場合には以下のような指定になります。
```zig
try writer.print("-{d:0>2}", .{self.month});
```

- `0`: `fill`に対応。format時に空いた空白スペースを埋める文字列を指定
- `>`: `alignment`に対応。`>`で右寄せ。
- `2`: `width`に対応。何文字表示するか。

## まとめ

Zigを触り始めたばかりですが、Zigの標準ライブラリ（`std`）を調べるときに、testが実装ファイルに書かれているので仕様がわかりやすくて良いです。ただ、ggってもわからないことが多く、公式ドキュメントやソースを参考にすることが多いです。


## 参考

- ソース：https://github.com/ziglang/zig
- 公式： https://ziglang.org/documentation/master/
- std：https://ziglang.org/documentation/master/std/#root
  - ソースに直に飛べるので地味に便利
- チュートリアル: https://ziglearn.org/chapter-0/
