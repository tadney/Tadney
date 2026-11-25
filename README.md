name: Update neon profile

on:
  schedule:
    - cron: "30 0,12 * * *"   # twice a day (UTC)
  workflow_dispatch:
  push:
    branches: [main]
    paths: [.github/workflows/profile.yml]

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build neon profile
        env:
          GH_USER: tadney
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          cat > "$RUNNER_TEMP/generate.py" <<'PYEOF'
          #!/usr/bin/env python3
          """Cyberpunk GitHub profile SVG generator (standard library only).

          Real data:  GH_USER=name GH_TOKEN=token python generate.py
          Preview:    python generate.py --demo
          """
          import datetime as dt
          import json
          import os
          import random
          import sys
          import urllib.request
          from xml.sax.saxutils import escape

          # ---- EDIT ME --------------------------------------------------------------
          TAGLINES = [
              "> initializing neural link...",
              "> full-stack netrunner // code is my cyberware",
              "> shipping commits from the neon underground",
              "> status: ONLINE. coffee levels: CRITICAL",
          ]
          # ---------------------------------------------------------------------------

          BG, PANEL, GRID = "#07030f", "#100824", "#2a1650"
          CYAN, PINK, YELLOW, GREEN = "#00f0ff", "#ff2bd6", "#fcee0a", "#39ff88"
          TEXT, DIM = "#e8e6ff", "#8d84b8"
          HEAT = ["#1a0f33", "#4a1a6e", "#9b1f9e", "#ff2bd6", "#00f0ff"]
          FONT = "font-family:'JetBrains Mono','Fira Code',Consolas,Menlo,monospace"
          W = 900


          # ---------------- data ----------------
          def gql(query, variables, token):
              req = urllib.request.Request(
                  "https://api.github.com/graphql",
                  data=json.dumps({"query": query, "variables": variables}).encode(),
                  headers={"Authorization": f"bearer {token}", "User-Agent": "neon-profile"},
              )
              with urllib.request.urlopen(req, timeout=60) as r:
                  out = json.load(r)
              if out.get("errors"):
                  raise SystemExit(f"GitHub API error: {out['errors']}")
              return out["data"]


          def fetch(user, token):
              years = gql("query($u:String!){user(login:$u){contributionsCollection{contributionYears}}}",
                          {"u": user}, token)["user"]["contributionsCollection"]["contributionYears"]
              q = """query($u:String!,$f:DateTime!,$t:DateTime!){user(login:$u){
                contributionsCollection(from:$f,to:$t){contributionCalendar{
                weeks{contributionDays{date contributionCount}}}}}}"""
              days = {}
              for y in years:
                  data = gql(q, {"u": user, "f": f"{y}-01-01T00:00:00Z", "t": f"{y}-12-31T23:59:59Z"}, token)
                  for week in data["user"]["contributionsCollection"]["contributionCalendar"]["weeks"]:
                      for d in week["contributionDays"]:
                          days[dt.date.fromisoformat(d["date"])] = d["contributionCount"]
              return days


          def demo_data(today):
              rng = random.Random(2077)
              days, d = {}, today - dt.timedelta(days=800)
              while d <= today:
                  busy = 0.8 if d.weekday() < 5 else 0.45
                  days[d] = int(rng.expovariate(1 / 5)) + 1 if rng.random() < busy else 0
                  d += dt.timedelta(days=1)
              for i in range(17):
                  days[today - dt.timedelta(days=i)] = rng.randint(1, 12)
              return days


          def fill(days, today):
              """Continuous date -> count mapping, no future dates, no gaps."""
              past = [d for d in days if d <= today]
              start = min(past) if past else today
              return {start + dt.timedelta(days=i): days.get(start + dt.timedelta(days=i), 0)
                      for i in range((today - start).days + 1)}


          def streaks(days):
              """Return list of (start, end, length) for every run of active days."""
              runs, start, prev = [], None, None
              for d in sorted(days):
                  if days[d] > 0:
                      start = start or d
                      prev = d
                  elif start:
                      runs.append((start, prev, (prev - start).days + 1))
                      start = None
              if start:
                  runs.append((start, prev, (prev - start).days + 1))
              return runs


          def current_streak(runs, today):
              # An empty "today" does not break the streak until the day is over.
              if runs and (today - runs[-1][1]).days <= 1:
                  return runs[-1]
              return None


          # ---------------- drawing helpers ----------------
          def text(x, y, s, size=12, fill=TEXT, anchor="start", extra=""):
              return (f'<text x="{x}" y="{y}" font-size="{size}" fill="{fill}" '
                      f'text-anchor="{anchor}" {extra}>{escape(str(s))}</text>')


          def rect(x, y, w, h, fill, extra=""):
              return f'<rect x="{x:.1f}" y="{y:.1f}" width="{w:.1f}" height="{h:.1f}" fill="{fill}" {extra}/>'


          def svg(h, body, style=""):
              return (f'<svg xmlns="http://www.w3.org/2000/svg" width="{W}" height="{h}" viewBox="0 0 {W} {h}">'
                      f'<style>text{{{FONT}}}{style}</style>{body}</svg>\n')


          def panel(h, title, accent=CYAN, demo=False):
              cut = 18
              out = (f'<path d="M1 1H{W-cut}L{W-1} {cut}V{h-1}H{cut}L1 {h-cut}Z" fill="{PANEL}" '
                     f'stroke="{accent}" stroke-width="1.5"/>')
              out += rect(16, 15, 5, 16, YELLOW)
              out += text(30, 28, title, 14, accent, extra='font-weight="bold" letter-spacing="2"')
              if demo:
                  out += text(W - 28, 28, "SAMPLE DATA", 12, YELLOW, "end", 'font-weight="bold"')
              return out


          def fmt(d):
              return d.strftime("%d %b %Y")


          # ---------------- animated banner ----------------
          def header(name):
              h, hz = 320, 190
              rng = random.Random(7)
              b = ('<defs><linearGradient id="sky" x1="0" y1="0" x2="0" y2="1">'
                   f'<stop offset="0" stop-color="{BG}"/><stop offset="1" stop-color="#2b0b4a"/></linearGradient>'
                   f'<clipPath id="frame"><rect width="{W}" height="{h}" rx="10"/></clipPath></defs>'
                   '<g clip-path="url(#frame)">')
              b += rect(0, 0, W, h, "url(#sky)")
              # data rain
              for i in range(46):
                  x, dur = i * 20 + 6, rng.uniform(3, 8)
                  col = rng.choice([CYAN, PINK, GREEN])
                  b += (f'<line x1="{x}" x2="{x}" y1="-40" y2="-10" stroke="{col}" stroke-width="1.5" opacity=".45">'
                        f'<animateTransform attributeName="transform" type="translate" from="0 0" to="0 {h+60}" '
                        f'dur="{dur:.1f}s" begin="-{rng.uniform(0, 8):.1f}s" repeatCount="indefinite"/></line>')
              # skyline with blinking windows
              x = 0
              while x < W:
                  bw, bh = rng.randint(30, 70), rng.randint(25, 90)
                  b += rect(x, hz - bh, bw - 3, bh, "#0c0620")
                  for _ in range(4):
                      b += (f'<rect x="{x + rng.randint(4, bw-12)}" y="{hz - rng.randint(8, bh-5)}" width="3" height="4" '
                            f'fill="{rng.choice([CYAN, YELLOW, PINK])}"><animate attributeName="opacity" '
                            f'values="1;.15;1" dur="{rng.uniform(1.5, 5):.1f}s" repeatCount="indefinite"/></rect>')
                  x += bw
              # moving perspective grid
              b += rect(0, hz, W, h - hz, BG)
              for i in range(-12, 13):
                  b += f'<line x1="{W/2 + i*18}" y1="{hz}" x2="{W/2 + i*110}" y2="{h}" stroke="{PINK}" opacity=".45"/>'
              for i in range(8):
                  b += (f'<line x1="0" x2="{W}" y1="0" y2="0" stroke="{PINK}">'
                        f'<animateTransform attributeName="transform" type="translate" '
                        f'values="0 {hz};0 {hz+12};0 {hz+45};0 {h}" keyTimes="0;.35;.7;1" dur="3.2s" '
                        f'begin="-{i*0.4:.1f}s" repeatCount="indefinite"/></line>')
              b += f'<line x1="0" x2="{W}" y1="{hz}" y2="{hz}" stroke="{CYAN}" stroke-width="2"/>'
              # glitch title
              size = min(54, int(780 / (max(len(name), 1) * 0.68)))
              for cls, col in (("g1", CYAN), ("g2", PINK), ("main", "#ffffff")):
                  b += f'<g class="{cls}">' + text(W / 2, 105, name.upper(), size, col, "middle",
                                                   'font-weight="bold" letter-spacing="3"') + "</g>"
              # typing lines, one after another
              n, slot = len(TAGLINES), 4.0
              total = n * slot
              for i, line in enumerate(TAGLINES):
                  width = len(line) * 9.8
                  x0 = W / 2 - width / 2
                  a, z = i / n, (i + 1) / n
                  times = [0, a, a + 0.6 / n, z - 0.001, z, 1]
                  vals = [0, 0, width, width, 0, 0]
                  kt = ";".join(f"{t:.4f}" for t in times)
                  v = ";".join(f"{q:.0f}" for q in vals)
                  b += (f'<clipPath id="t{i}"><rect x="{x0:.0f}" y="262" height="30" width="0">'
                        f'<animate attributeName="width" values="{v}" keyTimes="{kt}" dur="{total}s" '
                        f'repeatCount="indefinite"/></rect></clipPath>')
                  b += (f'<g clip-path="url(#t{i})">' +
                        text(f"{x0:.0f}", 283, line, 16, GREEN, extra=f'textLength="{width:.0f}"') + "</g>")
              # scanlines and sweep
              for y in range(0, h, 4):
                  b += rect(0, y, W, 1, "#000", 'opacity=".22"')
              b += (f'<rect width="{W}" height="40" fill="{CYAN}" opacity=".06"><animate attributeName="y" '
                    f'from="-40" to="{h}" dur="5s" repeatCount="indefinite"/></rect></g>')
              style = (".g1{animation:g1 2.8s infinite steps(1);opacity:.85}"
                       ".g2{animation:g2 2.8s infinite steps(1);opacity:.85}"
                       "@keyframes g1{0%{transform:translate(-2px,0)}82%{transform:translate(-9px,2px)}"
                       "86%{transform:translate(6px,-2px)}90%{transform:translate(-2px,0)}}"
                       "@keyframes g2{0%{transform:translate(2px,0)}82%{transform:translate(9px,-2px)}"
                       "86%{transform:translate(-6px,2px)}90%{transform:translate(2px,0)}}")
              return svg(h, b, style)


          def divider():
              b = rect(0, 0, W, 24, BG)
              b += f'<line x1="0" y1="12" x2="{W}" y2="12" stroke="{GRID}" stroke-width="2"/>'
              for col, begin in ((CYAN, "0s"), (PINK, "-1.5s")):
                  b += (f'<rect y="10" width="120" height="4" fill="{col}"><animate attributeName="x" from="-120" '
                        f'to="{W}" dur="3s" begin="{begin}" repeatCount="indefinite"/></rect>')
              return svg(24, b)


          # ---------------- static graphs ----------------
          def streak_cards(days, today, demo):
              runs = streaks(days)
              cur = current_streak(runs, today)
              best = max(runs, key=lambda r: r[2], default=None)
              total = sum(days.values())
              active = sum(1 for c in days.values() if c)
              peak = max(days, key=days.get)
              cards = [
                  ("CURRENT STREAK", f"{cur[2] if cur else 0}d", f"since {fmt(cur[0])}" if cur else "no active run", PINK),
                  ("LONGEST STREAK", f"{best[2] if best else 0}d", ("ongoing" if best == cur else f"ended {fmt(best[1])}") if best else "-", CYAN),
                  ("CONTRIBUTIONS", f"{total:,}", f"since {fmt(min(days))}", GREEN),
                  ("ACTIVE DAYS", f"{active:,}", f"{100 * active / len(days):.0f}% of all days", YELLOW),
                  ("BEST DAY", f"{days[peak]}", fmt(peak) if days[peak] else "-", PINK),
              ]
              b = panel(170, "// STREAK CORE", PINK, demo)
              for i, (label, value, sub, col) in enumerate(cards):
                  x = 16 + i * 175
                  b += rect(x, 46, 168, 108, BG, f'stroke="{col}" stroke-opacity=".6"')
                  b += text(x + 84, 68, label, 11, DIM, "middle")
                  b += text(x + 84, 112, value, 34, col, "middle", 'font-weight="bold"')
                  b += text(x + 84, 138, sub, 10, TEXT, "middle")
              return svg(170, b)


          def heatmap(days, today, demo):
              b = panel(190, "// CONTRIBUTION MATRIX - LAST 53 WEEKS", CYAN, demo)
              start = today - dt.timedelta(days=(today.weekday() + 1) % 7 + 52 * 7)  # a Sunday
              span = [start + dt.timedelta(days=i) for i in range((today - start).days + 1)]
              top = max((days.get(d, 0) for d in span), default=0) or 1
              month, last_label = None, -3
              for i, d in enumerate(span):
                  col, row = divmod(i, 7)
                  c = days.get(d, 0)
                  level = 0 if c == 0 else min(4, 1 + int(3.999 * c / top))
                  b += (f'<rect x="{52 + col*15.5:.1f}" y="{62 + row*14}" width="12" height="11" rx="2" '
                        f'fill="{HEAT[level]}"/>')
                  if row == 0 and d.month != month:
                      month = d.month
                      if col - last_label >= 3 and col < 51:
                          b += text(52 + col * 15.5, 54, d.strftime("%b"), 10, DIM)
                          last_label = col
              for row, lab in ((1, "Mon"), (3, "Wed"), (5, "Fri")):
                  b += text(18, 71 + row * 14, lab, 10, DIM)
              b += text(700, 175, "less", 10, DIM, "end")
              for i, c in enumerate(HEAT):
                  b += rect(708 + i * 16, 166, 12, 11, c)
              b += text(792, 175, "more", 10, DIM)
              return svg(190, b)


          def bars(b, values, x, y, w, h, color, labels):
              """Bar chart with y gridlines. labels: {bar index: text}."""
              top = max(values, default=0) or 1
              for k in (0, 0.5, 1):
                  gy = y + h - h * k
                  b.append(f'<line x1="{x}" x2="{x+w}" y1="{gy}" y2="{gy}" stroke="{GRID}"/>')
                  b.append(text(x - 6, gy + 3, f"{top * k:g}", 9, DIM, "end"))
              step = w / max(len(values), 1)
              for i, v in enumerate(values):
                  bh = h * v / top
                  b.append(rect(x + i * step + step * 0.15, y + h - bh, step * 0.7, bh, color(i, v)))
                  if i in labels:
                      b.append(text(x + i * step + step / 2, y + h + 16, labels[i], 9, DIM, "middle"))


          def daily(days, today, demo):
              span = [today - dt.timedelta(days=i) for i in range(89, -1, -1)]
              vals = [days.get(d, 0) for d in span]
              b = [panel(240, "// SIGNAL - DAILY CONTRIBUTIONS, LAST 90 DAYS", GREEN, demo)]
              bars(b, vals, 50, 55, 825, 145, lambda i, v: CYAN,
                   {i: span[i].strftime("%d %b") for i in range(4, 90, 14)})
              # 7-day rolling average
              pts = []
              for i, d in enumerate(span):
                  avg = sum(days.get(d - dt.timedelta(days=k), 0) for k in range(7)) / 7
                  pts.append(f"{50 + (i + .5) * 825 / 90:.1f},{200 - 145 * avg / (max(vals) or 1):.1f}")
              b.append(f'<polyline points="{" ".join(pts)}" fill="none" stroke="{PINK}" stroke-width="2"/>')
              b.append(text(450, 232, f"cyan = daily  |  pink = 7-day average  |  total {sum(vals):,}", 10, DIM, "middle"))
              return svg(240, "".join(b))


          def history(days, today, demo):
              year_ago = today - dt.timedelta(days=364)
              # running streak length per day
              run, lengths = 0, {}
              for d in sorted(days):
                  run = run + 1 if days[d] else 0
                  lengths[d] = run
              span = [year_ago + dt.timedelta(days=i) for i in range(365)]
              vals = [lengths.get(d, 0) for d in span]
              b = [panel(460, "// STREAK HISTORY - LAST 365 DAYS", PINK, demo)]
              bars(b, vals, 50, 55, 825, 140, lambda i, v: PINK,
                   {i: d.strftime("%b") for i, d in enumerate(span) if d.day == 1})
              b.append(text(30, 245, "TOP 5 STREAKS (ALL TIME)", 12, YELLOW, extra='font-weight="bold"'))
              top = sorted(streaks(days), key=lambda r: (r[2], r[1]), reverse=True)[:5]
              for i, (s, e, n) in enumerate(top):
                  y = 262 + i * 36
                  b.append(text(30, y + 15, f"#{i+1}", 12, DIM))
                  b.append(text(66, y + 15, f"{fmt(s)} - {fmt(e)}", 11))
                  b.append(rect(300, y + 3, 500, 16, BG))
                  b.append(rect(300, y + 3, 500 * n / top[0][2], 16, [CYAN, PINK, GREEN, YELLOW, DIM][i]))
                  b.append(text(870, y + 15, f"{n}d", 12, TEXT, "end"))
              return svg(460, "".join(b))


          def rhythm(days, today, demo):
              year = {d: c for d, c in days.items() if (today - d).days < 365}
              week = [sum(c for d, c in year.items() if d.weekday() == i) for i in range(7)]
              months = []
              y, m = today.year, today.month
              for _ in range(12):
                  months.insert(0, (y, m))
                  y, m = (y - 1, 12) if m == 1 else (y, m - 1)
              monthly = [sum(c for d, c in days.items() if (d.year, d.month) == k) for k in months]
              b = [panel(230, "// RHYTHM - WEEKDAY (365d) AND MONTHLY (12mo)", YELLOW, demo)]
              bars(b, week, 52, 60, 300, 130, lambda i, v: PINK if v == max(week) else CYAN,
                   dict(enumerate(["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"])))
              bars(b, monthly, 430, 60, 445, 130, lambda i, v: YELLOW if v == max(monthly) else GREEN,
                   {i: dt.date(y, m, 1).strftime("%b") for i, (y, m) in enumerate(months)})
              return svg(230, "".join(b))


          def main():
              demo = "--demo" in sys.argv
              today = dt.datetime.now(dt.timezone.utc).date()
              if demo:
                  name, raw = os.getenv("GH_USER") or "NETRUNNER", demo_data(today)
              else:
                  name, token = os.getenv("GH_USER"), os.getenv("GH_TOKEN")
                  if not name or not token:
                      raise SystemExit("Set GH_USER and GH_TOKEN, or run with --demo")
                  raw = fetch(name, token)
              days = fill(raw, today)
              os.makedirs("assets", exist_ok=True)
              files = {
                  "header": header(name),
                  "divider": divider(),
                  "streaks": streak_cards(days, today, demo),
                  "heatmap": heatmap(days, today, demo),
                  "daily": daily(days, today, demo),
                  "history": history(days, today, demo),
                  "rhythm": rhythm(days, today, demo),
              }
              for key, content in files.items():
                  with open(f"assets/{key}.svg", "w", encoding="utf-8") as f:
                      f.write(content)
              print(f"Wrote {len(files)} SVGs for {name}")


          if __name__ == "__main__":
              main()
          PYEOF
          python3 "$RUNNER_TEMP/generate.py"
          cat > README.md <<'MDEOF'
          <div align="center">

          <img src="./assets/header.svg" width="100%" alt="Animated cyberpunk banner" />

          <img src="./assets/divider.svg" width="100%" alt="" />

          <img src="./assets/streaks.svg" width="100%" alt="Current and longest streak, totals" />

          <img src="./assets/heatmap.svg" width="100%" alt="Contribution heatmap" />

          <img src="./assets/daily.svg" width="100%" alt="Daily contributions, last 90 days" />

          <img src="./assets/history.svg" width="100%" alt="Streak history and top streaks" />

          <img src="./assets/rhythm.svg" width="100%" alt="Weekday and monthly activity" />

          <img src="./assets/divider.svg" width="100%" alt="" />

          ### `> LOADOUT`

          ![Python](https://img.shields.io/badge/Python-07030f?style=for-the-badge&logo=python&logoColor=00f0ff)
          ![JavaScript](https://img.shields.io/badge/JavaScript-07030f?style=for-the-badge&logo=javascript&logoColor=fcee0a)
          ![React](https://img.shields.io/badge/React-07030f?style=for-the-badge&logo=react&logoColor=00f0ff)
          ![Git](https://img.shields.io/badge/Git-07030f?style=for-the-badge&logo=git&logoColor=ff2bd6)

          <img src="./assets/divider.svg" width="100%" alt="" />

          </div>
          MDEOF
      - name: Commit if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add assets README.md
          git diff --cached --quiet || git commit -m "chore: refresh neon stats"
          git push
