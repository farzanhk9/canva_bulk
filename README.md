#!/usr/bin/env python3
# File: canva_bulk_builder.py
# Build Canva Bulk Create CSV + IG captions from a simple product CSV
# No external dependencies. Python 3.9+

import argparse, csv, os, re, random, unicodedata
from datetime import datetime
from typing import List, Tuple

EMOJIS = {
    "friendly": ["✨","🔥","💥","💡","😍","⚡","🙌","🎯","🛒","🧠","⭐"],
    "luxury":   ["✨","💎","🏷️","🎩","🌙","🖤"],
    "tech":     ["🧠","⚙️","🚀","💡","📈","🔧"],
    "sport":    ["🏃","⚡","🔥","🏅","👟","💪"],
}

CTAS = [
    "Shop now", "Tap to see more", "DM us for details", "Save this for later",
    "Comment your favorite", "Tag a friend", "Link in bio"
]

def choose_emojis(tone: str, k: int=2) -> str:
    pool = EMOJIS.get(tone, EMOJIS["friendly"])
    return " ".join(random.sample(pool, k=min(k, len(pool))))

def slugify(text: str) -> str:
    text = unicodedata.normalize("NFKD", text).encode("ascii","ignore").decode("ascii", "ignore")
    text = re.sub(r"[^a-zA-Z0-9]+", "-", text).strip("-").lower()
    return text or "item"

def mix_hashtags(keywords: List[str], count: int=18) -> str:
    kw = [k.strip().lower().replace(" ","") for k in keywords if k.strip()]
    big = ["fashion","style","ootd","streetwear","sale","viral","reels","explore","trending","howto","unboxing","shopnow"]
    medium = [f"{c}", f"{c}daily", f"{c}guide", f"{c}tips" for c in kw]
    niche = []
    for c in kw:
        niche += [f"{c}community", f"{c}deals", f"{c}reviews", f"{c}hack"]
    pool = []
    pool += random.sample(big, k=min(6,len(big)))
    pool += random.sample(medium, k=min(6,len(medium)))
    pool += random.sample(niche, k=min(12,len(niche)))
    tags, seen = [], set()
    for t in pool:
        t = "#"+t.replace("#","")
        if t not in seen:
            tags.append(t); seen.add(t)
        if len(tags) >= count: break
    return " ".join(tags)

def caption_en(name, tone, audience, features, cta, tags) -> str:
    hook = {
        "friendly": f"{choose_emojis('friendly')} {name} you’ll actually love.",
        "luxury":   f"Crafted to be noticed: {name} {choose_emojis('luxury')}",
        "tech":     f"Smarter design, better results — {name} {choose_emojis('tech')}",
        "sport":    f"Built to move — {name} {choose_emojis('sport')}",
    }.get(tone, f"{choose_emojis('friendly')} {name} that hits different.")
    feats = "\n".join([f"• {f.strip().capitalize()}" for f in features[:3] if f.strip()])
    return f"""{hook}

{feats}

{cta} {choose_emojis(tone)}
{tags}""".strip()

def caption_fa(name, tone, audience, features, cta, tags) -> str:
    # ساده و کوتاه (RTL را به عهده‌ی شبکه می‌گذاریم)
    tone_emoji = choose_emojis(tone)
    feats = "\n".join([f"• {f.strip()}" for f in features[:3] if f.strip()])
    return f"""{tone_emoji} {name}

{feats}

{cta} {tone_emoji}
{tags}""".strip()

def title_sub_en(name, price, currency) -> Tuple[str,str]:
    return name, f"Only {currency_symbol(currency)}{price}"

def title_sub_fa(name, price, currency) -> Tuple[str,str]:
    return name, f"فقط {price} {currency}"

def currency_symbol(cur: str) -> str:
    cur = (cur or "").upper()
    return {"EUR":"€","USD":"$","GBP":"£","IRR":"﷼","TRY":"₺"}.get(cur, f"{cur} ")

def build_rows(record: dict, langs: List[str], max_tags: int):
    name = record.get("name","").strip()
    price = record.get("price","").strip()
    currency = record.get("currency","").strip() or "EUR"
    colors = [c.strip() for c in (record.get("colors","").split("|")) if c.strip()]
    sizes = record.get("sizes","").strip()
    keywords = [k.strip() for k in record.get("keywords","").split(",") if k.strip()]
    features = [f.strip() for f in record.get("features","").split(";") if f.strip()]
    audience = record.get("audience","everyone").strip()
    tone = (record.get("tone","friendly") or "friendly").lower()
    url = record.get("url","").strip()

    slug = slugify(name)
    filename = f"{slug}.jpg"

    tags = mix_hashtags(keywords, count=max_tags)
    cta_en = "Shop now"
    cta_fa = "همین حالا بخر"

    out_rows = []
    if "en" in langs:
        title, subtitle = title_sub_en(name, price, currency)
        cap = caption_en(name, tone, audience, features, cta_en, tags)
        out_rows.append({
            "Lang":"en","Title":title,"SubTitle":subtitle,"Price":f"{currency_symbol(currency)}{price}",
            "Bullet1":(features[0] if len(features)>0 else ""),
            "Bullet2":(features[1] if len(features)>1 else ""),
            "Bullet3":(features[2] if len(features)>2 else ""),
            "Colors":" / ".join(colors), "Sizes":sizes,
            "CTA":cta_en, "Hashtags":tags, "URL":url,
            "FileName":filename, "Slug":slug
        })
    if "fa" in langs:
        title, subtitle = title_sub_fa(name, price, currency)
        cap = caption_fa(name, tone, audience, features, cta_fa, tags)
        out_rows.append({
            "Lang":"fa","Title":title,"SubTitle":subtitle,"Price":f"{price} {currency}",
            "Bullet1":(features[0] if len(features)>0 else ""),
            "Bullet2":(features[1] if len(features)>1 else ""),
            "Bullet3":(features[2] if len(features)>2 else ""),
            "Colors":" / ".join(colors), "Sizes":sizes,
            "CTA":cta_fa, "Hashtags":tags, "URL":url,
            "FileName":filename, "Slug":slug
        })
    return out_rows

def main():
    ap = argparse.ArgumentParser(description="Canva Bulk Create builder + IG captions from products CSV")
    ap.add_argument("--input", required=True, help="products.csv")
    ap.add_argument("--outdir", default="out")
    ap.add_argument("--lang", default="en,fa", help="comma-separated languages (en,fa)")
    ap.add_argument("--max-tags", type=int, default=18)
    args = ap.parse_args()

    langs = [x.strip().lower() for x in args.lang.split(",") if x.strip()]
    os.makedirs(args.outdir, exist_ok=True)

    rows_out = []
    captions = []

    with open(args.input, "r", encoding="utf-8-sig") as f:
        reader = csv.DictReader(f)
        for rec in reader:
            name = rec.get("name","").strip()
            price = rec.get("price","").strip()
            currency = rec.get("currency","").strip() or "EUR"
            keywords = [k.strip() for k in rec.get("keywords","").split(",") if k.strip()]
            features = [f.strip() for f in rec.get("features","").split(";") if f.strip()]
            url = rec.get("url","").strip()
            tone = (rec.get("tone","friendly") or "friendly").lower()

            bundles = build_rows(rec, langs, args.max_tags)
            rows_out.exten_

