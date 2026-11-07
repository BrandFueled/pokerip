[README.md](https://github.com/user-attachments/files/34521873/README.md)
# POKÉ RIP

A for-fun Pokémon TCG pack-opening simulator. Rip packs, chase hits, grade slabs, build master sets, and get roasted by Kyle. **Fake money only.** No purchases, no cash-outs, no real prizes.

**Play it:** https://brandfueled.github.io/pokerip/

## What's in here

| File | What it is |
|---|---|
| `index.html` | The whole game. HTML, CSS and JavaScript in one file. No build step, no framework. |
| `pack.webp` | Pack art (Booster, Hit, Vintage, 1st Edition, Mystery) |
| `single.webp` | Single-card art. Also the back of every card. |

## How it works

- **Card data and prices** come from the free [Pokémon TCG API](https://pokemontcg.io) (TCGplayer market prices). Sets are cached in the browser for 3 days. If the API is down, the game falls back to a small built-in sample pool.
- **Saves** live in the browser (`localStorage`). If a player logs in, saves also sync to **Supabase**, which powers cross-device play, the leaderboard and the live pull feed.
- **Hosting** is GitHub Pages. Anything committed to `main` goes live in 1 to 2 minutes.

## Editing

1. Open `index.html` in any editor (VS Code is great).
2. Almost everything you'd want to tweak is a constant near the top of the `<script>` section (search for these names):

| Want to change… | Look for |
|---|---|
| How lucky the whole game feels | `LUCK` (default 4, higher = more hits) |
| Pack prices, contents, unlock levels | `TIERS` |
| Value odds for each pack slot | `RARE_BANDS`, `HIT_BANDS`, `CHASE_BANDS`, etc. Format is `[min $, max $, weight %]` |
| Slab Grab, Daily Spin, God Tier | `MODES` |
| Mystery Pack outcomes | `MYSTERY` |
| Grade multipliers and grading odds | `GRADE_MULT`, `SUBMIT_GRADES`, `GRADE_FEE` |
| Levels and level-up rewards | `XP_AT`, `LVL_REWARDS`, `UNLOCKS` |
| Trophy Case (the end game) | `TROPHIES` |
| Featured master sets | `MASTER` |
| Default set rotation | `DEFAULT_SETS`, `VINTAGE_SETS` |
| Starting cash, tickets, markups | `START`, `TICKET_PRICE`, `BUY_MARKUP`, `TRADE_BONUS`, `VAULT_MAX` |
| **Roasts** (the important part) | `ROAST` (random lines) and `contextRoasts()` (lines based on a player's own stats) |
| Badges | `MILESTONES` |
| Bailout jokes | `BAIL_LINES` |
| Kyle's weekly bounties | `makeBounties()` |

3. Test it locally (see below), then commit to `main`.

**Adding roasts:** just add a string to the right list in `ROAST` (`awful`, `meh`, `good`, `huge`, `grade`, `grailMiss`, `grailWin`). Watch your quotes: if a line has an apostrophe, wrap it in double quotes, like `"Kyle's mad."`.

## Testing locally

Opening the file directly sometimes blocks the card API, so run a tiny local server from the repo folder:

```bash
python3 -m http.server 8000
```

Then open http://localhost:8000. Test on your phone too. **Mobile first:** most players are on iPhone Safari.

To quickly check for a JavaScript typo: open the browser console (right-click → Inspect → Console) and look for red errors.

Handy console tricks while testing (fake money, so go wild):

```js
game.cash = 10000; save();            // more money
game.xp = 10000; save();              // jump levels
gradingCards().forEach(c => c.grading = Date.now()); save();  // finish grading now
```

## Supabase (logins, leaderboard, live feed)

The keys in `index.html` (`SUPA_URL`, `SUPA_KEY`) are the **publishable** key, which is safe to be public. **Never** put the secret key in this repo.

Settings that must stay this way in Supabase:
- **Authentication → Sign In / Providers → Email:** provider **ON**, "Confirm email" **OFF**. Logins are username-only. Behind the scenes the game uses `pokerip.<username>@brandfueled.com` as the email.

Tables (already created). Here's the SQL in case it ever needs to be rebuilt:

```sql
create table public.saves (
  user_id uuid primary key references auth.users on delete cascade,
  username text unique not null,
  data jsonb not null,
  networth numeric default 0,
  level int default 1,
  trophies int default 0,
  prestige int default 0,
  best_value numeric default 0,
  best_name text,
  shame int default 0,
  updated_at timestamptz default now()
);
alter table public.saves enable row level security;
create policy "anyone can read" on public.saves for select using (true);
create policy "insert own" on public.saves for insert with check (auth.uid() = user_id);
create policy "update own" on public.saves for update using (auth.uid() = user_id);

create table public.pulls (
  id bigint generated always as identity primary key,
  user_id uuid references auth.users on delete cascade,
  username text not null,
  name text not null,
  value numeric not null,
  img text, grade int, first boolean default false, source text,
  created_at timestamptz default now()
);
alter table public.pulls enable row level security;
create policy "anyone can read pulls" on public.pulls for select using (true);
create policy "post own pulls" on public.pulls for insert with check (auth.uid() = user_id);
```

## Code map

The script is one long file, in this order:

config → helpers → sound + effects → card catalog (API loading) → game state and saving → levels, badges, grading, Grail Quest, bounties → pull generation → views (Shop, Binder, Collect, Slab Grab, Stats) → rip flow → card viewer → Slab Grab reel → share images → cloud saves and feed → bailouts and odds screens → button actions (`ACTIONS`) → boot.

Buttons use `data-action="something"`, and the matching function lives in the `ACTIONS` object near the bottom.

## House rules

- **Keep it fun.** When in doubt, roast harder and reward more.
- **Mobile first.** Check every change at phone size.
- **Keep the jokes:** "O no, you have a gambling problem!", alert Kyle, the bailout limit, "go outside."
- **No real money, ever.** Everything stays fake.
- **No official branding** on slabs or pack names. Slabs say "POKÉ RIP GRADED". Real card images come from the API.
- **iPhone gotchas already solved** (don't break them): no animations or filters inside the card flip faces, selects use a 16px font so iOS doesn't zoom, and the bottom nav gets nudged back into place after the keyboard closes.
