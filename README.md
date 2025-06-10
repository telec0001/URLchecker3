# URLchecker3

# koushin.py
import streamlit as st
import requests
import hashlib
import json
import os
import datetime
import re
from bs4 import BeautifulSoup

# 作業フォルダとデータファイルのパス
WORK_DIR = r"C:\Users\NISHIUMATatsuya\OneDrive - GSS\デスクトップ\python"
os.makedirs(WORK_DIR, exist_ok=True)
DATA_FILE = os.path.join(WORK_DIR, "url_data.json")

def load_data():
    """旧フォーマットにも対応して新フォーマットにマイグレーション"""
    if not os.path.exists(DATA_FILE):
        return {}

    raw = json.load(open(DATA_FILE, 'r', encoding='utf-8'))
    migrated = False
    data = {}

    for url, info in raw.items():
        if isinstance(info, str):
            # 旧フォーマット detected
            data[url] = {
                "hash": info,
                "last_checked": "N/A",
                "initialized": False
            }
            migrated = True
        else:
            data[url] = info

    if migrated:
        with open(DATA_FILE, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)

    return data

def save_data(d):
    with open(DATA_FILE, 'w', encoding='utf-8') as f:
        json.dump(d, f, ensure_ascii=False, indent=2)

def fetch_hash(url: str) -> str:
    """
    ・マカオ新聞トップページの場合 → 「カジノ・IR」セクションのみ抽出してMD5
    ・それ以外 → Last-Modified か本文比較モード
    """
    # 1) マカオ新聞トップならセクション抽出
    if url.rstrip('/') == "https://www.macaushimbun.com":
        resp = requests.get(url, timeout=10)
        resp.encoding = resp.apparent_encoding
        soup = BeautifulSoup(resp.text, "html.parser")

        # h3見出しで「カジノ・IR」を探す
        header = soup.find(lambda tag:
                           tag.name in ("h2","h3","h4") and
                           "カジノ・IR" in tag.get_text())
        if header:
            section_texts = []
            for sib in header.next_siblings:
                # 同じレベルの次の見出しが来たら区切り終了
                if getattr(sib, "name", None) in ("h2","h3","h4"):
                    break
                txt = sib.get_text(separator="\n", strip=True)
                if txt:
                    section_texts.append(txt)
            combined = "\n".join(section_texts)
            return hashlib.md5(combined.encode("utf-8")).hexdigest()
        # 見つからなければフォールバックに落ちる

    # 2) HEADでLast-Modifiedチェック
    try:
        head = requests.head(url, allow_redirects=True, timeout=5)
        lm = head.headers.get("Last-Modified")
        if lm:
            return "LM:" + lm
    except Exception:
        pass

    # 3) 本文比較モード（不要タグを除去）
    resp = requests.get(url, timeout=10)
    resp.encoding = resp.apparent_encoding
    soup = BeautifulSoup(resp.text, "html.parser")
    for tag in soup(["script", "style", "header", "footer", "aside", "noscript"]):
        tag.decompose()
    main = soup.find("main") or soup.body or soup
    text = main.get_text(separator="\n", strip=True)
    return hashlib.md5(text.encode("utf-8")).hexdigest()

def main():
    st.title("URL 更新チェッカー（「カジノ・IR」対応版）")
    data = load_data()

    # ── URL一括追加 ──
    urls_input = st.text_area("URLを改行区切りで入力", height=120)
    if st.button("URLを一括追加"):
        now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        added = 0
        for u in filter(None, (u.strip() for u in urls_input.splitlines())):
            if u not in data:
                h = fetch_hash(u)
                data[u] = {"hash": h, "last_checked": now, "initialized": False}
                added += 1
        save_data(data)
        st.success(f"{added} 件を登録しました。次回チェックから比較します。")

    st.markdown("---")

    # ── 更新チェック ──
    if st.button("更新をチェック"):
        new_data = {}
        for u, info in data.items():
            old_h = info["hash"]
            h = fetch_hash(u)
            now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")

            if not info.get("initialized", False):
                st.write(f"- {u} ： **初回登録**（基準設定：{info['last_checked']}）")
            else:
                status = "更新あり" if h != old_h else "変更なし"
                st.write(f"- {u} ： **{status}** （前回：{info['last_checked']}）")

            new_data[u] = {"hash": h, "last_checked": now, "initialized": True}

        save_data(new_data)
        data = new_data  # UI表示用に更新

    st.markdown("---")
    st.subheader("登録済みURLと最終チェック日時")
    for u, info in data.items():
        st.write(f"- {u} ： 最終チェック {info['last_checked']}")

if __name__ == "__main__":
    main()
