# EKOR TI PAA - Laravel MVP

## Voting + Ticketing + Team Raffles (Final Dev Spec)

### Objective

Deliver a fully working MVP **before Nov 12 (1 week)** that lets users:

1. **Buy tickets/packs** tied to specific raffles (teams + prizes).
2. Automatically **earn voting points** per ticket purchase.
3. **Vote** for teams in active battles (1 point = 1 vote).
4. Automatically **run raffles at each deadline** and publish winners.
5. Maintain **live scoreboards**, anti-fraud controls, and daily reconciliation.



## Tech Stack

* **Laravel 11 (PHP 8.2+)**
* **MySQL 8 / MariaDB 10.6+**
* **Redis** (for cache + queue + scoreboard)
* **Laravel Horizon** (queue monitoring)
* **Filament Admin** (panel + CRUD + dashboard)
* **Moneroo payment gateway** (for now) → webhooks; Mobile Money
* **RateLimiter + reCAPTCHA v3** for anti-spam
* **Telescope** (staging), logs centralized in production
* **Cron + Supervisor/Horizon** for background jobs



## Main Entities (Eloquent Models)

### users

`id uuid | name | email | phone | password | points_balance:int | timestamps`

### teams

`id uuid | name | country | bio | media (json) | is_active bool`

### polls (battles)

`id uuid | title | start_at | end_at | status:draft/live/closed | timestamps`

### poll_team (pivot)

`id | poll_id | team_id | order`

### votes

`id uuid | user_id | poll_id | team_id | source (points/ticket/sms) | cost_points:int | timestamps`

### products (tickets/packs)

`id uuid | name | type (ticket/pack) | points | price_cfa | active bool`

### orders

`id uuid | user_id | product_id | amount_cfa | status (pending/paid/cancelled) | provider (stripe/paypal/mm/manual) | payment_ref | timestamps`

### transactions (ledger)

`id uuid | user_id | type (credit/debit) | amount | balance_after | source (purchase/vote/admin) | ref_id | timestamp`

### raffles (new for lottery)

`id uuid | team_id | title | description | ticket_price_cfa | points_per_ticket | winners_count | deadline_at | status (draft/live/drawing/drawn) | timestamps`

### raffle_purchases

`id uuid | raffle_id | user_id | order_id | quantity | price_cfa | range_start | range_end | created_at`

### raffle_results

`id uuid | raffle_id | winner_user_id | winning_ticket_no | created_at`

### webhooks_log

`id | event_id unique | provider | payload json | signature | processed bool | created_at`



## Raffle System (Team Lotteries)

### Concept

* Each **team** can run multiple **raffles**, each with its own prize, ticket price, and draw deadline.
* Buying a ticket gives the user:

  * voting points (`points_per_ticket × quantity`)
  * automatic entry into that raffle.
* When the deadline passes, a **scheduled job** picks random winners and stores them in `raffle_results`.

### Ticket Allocation Logic

Use **continuous ranges** per purchase (efficient).
Example: user A buys 3 tickets → range 1-3; user B buys 2 → 4-5.

On payment confirmation:

```php
$lastEnd = RafflePurchase::where('raffle_id',$raffle->id)->max('range_end') ?? 0;
$start = $lastEnd + 1;
$end   = $lastEnd + $quantity;

RafflePurchase::create([...range_start=>$start, range_end=>$end]);
$user->increment('points_balance', $quantity * $raffle->points_per_ticket);
Transaction::create([...]);
```

### Automatic Draw Algorithm

```php
$total = RafflePurchase::where('raffle_id',$raffle->id)->max('range_end');
for($i=0; $i<$raffle->winners_count; $i++){
   do { $r = random_int(1,$total); } while(in_array($r,$used));
   $purchase = RafflePurchase::where('raffle_id',$raffle->id)
                 ->where('range_end','>=',$r)
                 ->orderBy('range_end')->first();
   RaffleResult::create([
     'raffle_id'=>$raffle->id,
     'winner_user_id'=>$purchase->user_id,
     'winning_ticket_no'=>$r
   ]);
}
$raffle->update(['status'=>'drawn']);
```

* Use `random_int()` (CSPRNG) for fairness.
* Log each draw event for audit.

### Scheduler / Command

`php artisan raffles:draw`

```php
public function handle(){
  $raffles = Raffle::where('status','live')
            ->where('deadline_at','<=',now())->get();
  foreach($raffles as $r){
    $lock = Cache::lock("raffle:{$r->id}:draw",60);
    if(!$lock->get()) continue;
    try{
      $r->update(['status'=>'drawing']);
      $this->drawWinners($r);
      $r->update(['status'=>'drawn']);
    } finally { $lock->release(); }
  }
}
```

Cron job: `* * * * * php /artisan raffles:draw --quiet`



## Voting Flow

1. User logs in / registers.
2. Selects team and raffle ticket type.
3. Checkout → payment → webhook → points credited + raffle entries created.
4. Votes on active polls (`1 point = 1 vote`).
5. At deadline, raffle job runs → winners saved → visible on results page.



## API Routes (Examples)

### Public

`GET /raffles` → list all live raffles with team info + deadline
`GET /raffles/{id}` → details + countdown
`GET /raffles/{id}/results` → winners after draw
`GET /polls/live`, `GET /polls/{id}`, `GET /scoreboard/{poll}`

### Auth

`POST /register`, `POST /login`, `POST /logout`

### Checkout

`POST /raffles/{id}/checkout {quantity, provider}`
→ creates order, returns `payment_url`.
Webhook (`POST /webhook/payment/{provider}`) → marks order paid → creates `raffle_purchase` ranges + credits points.

### Voting

`POST /vote {poll_id, team_id}`

* Validates poll status & points.
* Atomic transaction → debit 1 point → insert vote → `Redis::INCR(score:poll:{id}:team:{id})`.
* Response: `{ok:true, balance, scoreboard}`.
* If insufficient points: HTTP 402 + `purchase_url`.



## Frontend / UI

* **Landing Page:** intro, CTA “Buy Ticket / Vote Now”.
* **Team Page:** team info + list of active raffles (cards: title, price, deadline).
* **Raffle Detail:** countdown, purchase form (quantity + payment method), rules.
* **Vote Page:** battle cards + progress bars (live percentages).
* **Results Page:** winner list (auto-updated after draw).
* **Admin Panel (Filament):**

  * CRUD for Teams, Polls, Raffles, Products, Orders, Users.
  * Dashboard cards (tickets sold, total votes, points credited).
  * “Force Draw” button + export CSV.



## Security / Fairness

* **reCAPTCHA v3** on /checkout and /vote.
* **Rate limit:** 20 votes/min per user (IP + auth).
* **Webhook signatures:** HMAC SHA256 + timestamp (+/- 5 min).
* **Idempotence:** store `event_id` in webhooks_log.
* **Random fairness:** use `random_int()`; optionally store `server_seed_hash` for audit.
* **HTTPS required.**



## Scoreboard (Live Votes)

* On each vote: `Redis::INCR("score:poll:{$poll->id}:team:{$team->id}")`.
* Endpoint `GET /scoreboard/{poll}` returns:

  ```json
  {
    "poll_id":"...",
    "total":1234,
    "teams":[{"team_id":"...","name":"...","votes":456,"pct":36.9}],
    "updated_at":"2025-11-05T20:00:00Z"
  }
  ```



## Admin Workflow (Create a Raffle)

1. Go to Admin > Raffles > “Add New”.
2. Select team → enter title (e.g., “iPhone 13 Giveaway”).
3. Set ticket price, points per ticket, number of winners, deadline.
4. Save → status = **Live**.
5. System handles purchases + entries + auto draw at deadline.



## Testing Checklist

* ✅ Payment Webhook → credits points + creates raffle_purchase ranges.
* ✅ Vote → debits 1 point + updates Redis scoreboard.
* ✅ Draw Command → creates RaffleResult rows correctly.
* ✅ Multiple winners → unique tickets only.
* ✅ No duplicate credits on replayed webhooks.
* ✅ Load test 500–1000 purchases + votes without errors.
* ✅ Reconciliation command verifies `credits = debits + balances`.



## One-Week Timeline (Deliver by Nov 12)

| Day               | Tasks                                                                                  |
| ----------------- | -------------------------------------------------------------------------------------- |
| **Day 1 (Nov 3)** | Set up Laravel project, .env, DB, Redis, migrations + models + seed (teams, products). |
| **Day 2**         | Auth (Breeze), basic UI pages + Filament Admin install.                                |
| **Day 3**         | Payment integration **Moneroo payment gateway**; Mobile Money + webhook credit logic.  |
| **Day 4**         | Voting endpoint + Redis scoreboard + rate-limit + reCAPTCHA.                           |
| **Day 5**         | Raffles module (models, CRUD, purchases, ranges, auto-draw command).                   |
| **Day 6**         | Results UI + email notifications + admin exports.                                      |
| **Day 7 (Nov 9)** | Full E2E testing, perf tuning, deployment prep.                                        |

*(Nov 10-12 reserved for real data + communication + final checks.)*

---

## Deliverables

* Complete Laravel project (prod-ready).
* Seed data (teams, raffles, test poll).
* Admin panel + live dashboard.
* Payment webhooks with HMAC check.
* Artisan command `raffles:draw` (auto scheduler).
* Postman collection (API tests).
* README with setup & cron/Horizon instructions.

