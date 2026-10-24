#!/usr/bin/env python3
"""把 events.m3u8 转成 TXT(名称,链接),写入 tv.txt 的「国外体育,#genre#」分组。
每次运行只替换标记之间的自动区块,手动添加的频道不会被改动。"""
import os
import re
import sys
import urllib.request

SRC = os.environ.get(
    "SRC_URL",
    "https://raw.githubusercontent.com/doms9/iptv/refs/heads/default/M3U8/events.m3u8",
)
TARGET = os.environ.get("TARGET_FILE", "tv.txt")
GENRE = os.environ.get("GENRE_LINE", "国外体育,#genre#")
# 1 = 把 #EXTVLCOPT 里的 Referer / User-Agent 以 url|Referer=..&User-Agent=.. 形式附在链接后
KEEP_HEADERS = os.environ.get("KEEP_HEADERS", "1") == "1"

START = "#AUTO-EVENTS-START"
END = "#AUTO-EVENTS-END"


def fetch(src: str) -> str:
    if os.path.exists(src):
        with open(src, encoding="utf-8-sig") as f:
            return f.read()
    req = urllib.request.Request(src, headers={"User-Agent": "Mozilla/5.0"})
    with urllib.request.urlopen(req, timeout=30) as r:
        return r.read().decode("utf-8-sig", errors="replace")


def parse_m3u(text: str):
    items, name, opts = [], None, {}
    for raw in text.splitlines():
        line = raw.strip()
        if not line:
            continue
        if line.startswith("#EXTINF"):
            name = line.split(",", 1)[1].strip() if "," in line else ""
            opts = {}
        elif line.startswith("#EXTVLCOPT:"):
            k, _, v = line[len("#EXTVLCOPT:"):].partition("=")
            opts[k.strip().lower()] = v.strip()
        elif line.startswith("#"):
            continue
        elif name is not None:
            url = line
            if KEEP_HEADERS:
                h = []
                if opts.get("http-referrer") or opts.get("http-referer"):
                    h.append("Referer=" + (opts.get("http-referrer") or opts.get("http-referer")))
                if opts.get("http-user-agent"):
                    h.append("User-Agent=" + opts["http-user-agent"])
                if h:
                    url += "|" + "&".join(h)
            name = name.replace(",", " ").strip() or "未命名"
            items.append((name, url))
            name, opts = None, {}
    return items


def main():
    items = parse_m3u(fetch(SRC))
    if not items:
        print("源里没有解析到频道,保持原文件不变。")
        return 0

    with open(TARGET, encoding="utf-8-sig") as f:
        text = f.read()
    nl = "\r\n" if "\r\n" in text else "\n"
    lines = text.splitlines()

    # 先去掉旧的自动区块
    if START in lines and END in lines:
        s, e = lines.index(START), lines.index(END)
        del lines[s:e + 1]

    # 找到分组
    try:
        g = next(i for i, l in enumerate(lines) if l.strip() == GENRE)
    except StopIteration:
        # 分组不存在就在文件末尾新建
        if lines and lines[-1].strip():
            lines.append("")
        lines.append(GENRE)
        g = len(lines) - 1
    end = g + 1
    while end < len(lines) and not lines[end].strip().endswith("#genre#"):
        end += 1
    # 插在分组最后一个非空行之后
    ins = end
    while ins > g + 1 and not lines[ins - 1].strip():
        ins -= 1

    existing = {l.split(",", 1)[1].strip() for l in lines[g + 1:ins] if "," in l}
    block = [START]
    seen = set()
    for name, url in items:
        if url in existing or url in seen:
            continue
        seen.add(url)
        block.append(f"{name},{url}")
    block.append(END)

    lines[ins:ins] = block
    new = nl.join(lines) + nl
    if new == text:
        print("无变化。")
        return 0
    with open(TARGET, "w", encoding="utf-8", newline="") as f:
        f.write(new)
    print(f"已更新:{len(block) - 2} 个频道写入「{GENRE}」。")
    return 0


if __name__ == "__main__":
    sys.exit(main())
