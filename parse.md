import re, sys, json

HEADER = re.compile(r'^(.*?)\s*-\s*August 9, 2026\s*-\s*Race\s+(\d+)\s*$')
ODDS = re.compile(r'(\d+\.\d{2})(\*?)')

def clean_track(s):
    return re.sub(r'[^A-Za-z ]', '', s).replace(' ', '')

def parse(path):
    lines = open(path, encoding='utf-8', errors='replace').read().split('\n')
    races = []
    cur = None
    in_table = False
    for ln in lines:
        raw = ln.rstrip()
        s = raw.strip()
        m = HEADER.match(s)
        if m and len(s) < 60:
            cur = {'track': clean_track(m.group(1)), 'race': int(m.group(2)),
                   'dist': None, 'surface': None, 'runners': []}
            races.append(cur)
            in_table = False
            continue
        if cur is None:
            continue
        if s.startswith('Distance:') and cur['dist'] is None:
            d = s.split('Current Track Record')[0].replace('Distance:', '').strip()
            cur['dist'] = d
            dl = d.lower()
            cur['surface'] = ('turf' if 'turf' in dl else
                              'aw' if ('all weather' in dl or 'tapeta' in dl) else 'dirt')
            continue
        if s.startswith('Last Raced'):
            in_table = True
            continue
        if in_table:
            if not s or s.startswith('Fractional Times') or s.startswith('Split Times'):
                in_table = False
                continue
            if '(' not in s:
                continue
            head = s.split('(')[0].strip()
            toks = head.split()
            if not toks:
                continue
            i = 1 if toks[0].startswith('---') else 2
            if len(toks) < i + 2:
                continue
            pgm = toks[i]
            name = ' '.join(toks[i + 1:]).strip()
            dq = name.upper().startswith('DQ-') or pgm.upper().startswith('DQ-')
            name = re.sub(r'^DQ-', '', name)
            odds_hits = ODDS.findall(s)
            if not odds_hits:
                continue
            val, star = odds_hits[-1]
            cur['runners'].append({
                'pos': len(cur['runners']) + 1,
                'pgm': pgm, 'name': name,
                'odds': float(val), 'fav': star == '*', 'dq': dq})
    return races

if __name__ == '__main__':
    out = []
    for p in sys.argv[1:]:
        out.extend(parse(p))
    json.dump(out, open('/home/claude/races.json', 'w'), indent=1)
    for r in out:
        favs = [x['name'] for x in r['runners'] if x['fav']]
        print(f"{r['track']} R{r['race']:>2} {r['surface']:5} n={len(r['runners']):2} "
              f"win={r['runners'][0]['name'][:22]:24} {r['runners'][0]['odds']:6.2f} "
              f"fav={favs[0][:20] if favs else 'NONE'}")
