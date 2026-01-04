# Customer Support Ticket REST API megvalósítása Laravel + JWT környezetben

**base_url:**  
`http://127.0.0.1:8000/api`

Az API egy egyszerű ügyfélszolgálati jegykezelő rendszert valósít meg. A backend célja a felhasználók, ticketek és ticket válaszok kezelése, valamint a felhasználói authentikáció JWT alapú Bearer token védelemmel. A struktúra és a dokumentáció felépítése a Payment Platform REST API példához hasonló.

**Funkciók:**

- **Authentikáció** (regisztráció, bejelentkezés, token kezelés JWT-vel)
- **Felhasználók** jegyek (tickets) létrehozása
- **Ticket CRUD** (Create, Read, Update, Delete – soft delete-tel)
- **Ticket válaszok kezelése** (ticket_replies)
- **Soft Delete támogatás** ticketekre és válaszokra – törölt adatok visszaállíthatók
- **Teszt adatok**:
  - 1 admin felhasználó (admin@example.com / `Admin_Secret_Pw2026!`)
  - 10 faker által generált “user” szerepkörű felhasználó magyar nevekkel
  - Minden felhasználóhoz 1–5 ticket
  - Minden tickethez 0–5 válasz, véletlenszerűen felhasználó vagy admin által

Az adatbázis neve: `supportPlatform`

---

## Adatbázis struktúra

Az alkalmazás adatbázisa három fő táblából áll: felhasználók (`users`), jegyek (`tickets`) és jegy válaszok (`ticket_replies`).  
Kapcsolatok:

- Egy felhasználóhoz több ticket tartozhat
- Egy tickethez több válasz tartozhat
- Minden válasz egy tickethez és egy felhasználóhoz kapcsolódik  

Az adatbázis MySQL-t használ, Laravel migrációkkal felépítve, foreign key megkötésekkel biztosítva az adatintegritást, hasonlóan a payment példához.

### Users tábla

A felhasználók alapadatait tárolja.

| Mező              | Típus          | Leírás                          |
|-------------------|----------------|----------------------------------|
| id                | bigint         | Elsődleges kulcs                |
| name              | varchar(255)   | Felhasználó neve                |
| email             | varchar(255)   | Email cím (egyedi)              |
| password          | varchar(255)   | Hash-elt jelszó                 |
| role              | varchar(50)    | Szerepkör (`user`, `admin`)     |
| email_verified_at | timestamp      | Email megerősítés időpontja     |
| created_at        | timestamp      | Létrehozás dátuma               |
| updated_at        | timestamp      | Utolsó módosítás dátuma         |

### Tickets tábla

Ticketek tárolása felhasználókhoz kapcsolva.  
**Soft Delete támogatással** – a törölt rekordok fizikailag megmaradnak az adatbázisban.

| Mező        | Típus        | Leírás                                                |
|-------------|--------------|--------------------------------------------------------|
| id          | bigint       | Elsődleges kulcs                                      |
| user_id     | bigint       | Foreign key (users.id) – a ticket tulajdonosa         |
| subject     | varchar(255) | Ticket tárgya                                         |
| description | text         | Ticket leírása                                        |
| priority    | enum         | `low`, `medium`, `high`                               |
| status      | enum         | `open`, `in_progress`, `closed`                       |
| created_at  | timestamp    | Létrehozás dátuma                                     |
| updated_at  | timestamp    | Utolsó módosítás dátuma                               |
| deleted_at  | timestamp    | Soft delete – törlés dátuma (nullable)                |

**Kapcsolat:** `belongsTo(User)`, `hasMany(TicketReply)`

### Ticket_replies tábla

Válaszok tárolása ticketekhez kapcsolva.  
**Soft Delete támogatással** – a törölt rekordok fizikailag megmaradnak az adatbázisban.

| Mező       | Típus      | Leírás                                        |
|------------|------------|-----------------------------------------------|
| id         | bigint     | Elsődleges kulcs                              |
| ticket_id  | bigint     | Foreign key (tickets.id)                      |
| user_id    | bigint     | Foreign key (users.id) – ki írta a választ    |
| message    | text       | Válasz szövege                                |
| created_at | timestamp  | Létrehozás dátuma                             |
| updated_at | timestamp  | Utolsó módosítás dátuma                       |
| deleted_at | timestamp  | Soft delete – törlés dátuma (nullable)        |

**Kapcsolat:** `belongsTo(Ticket)`, `belongsTo(User)`

---

## Eloquent modellek és kapcsolatok

### User model (`app/Models/User.php`)

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Notifications\Notifiable;
use Tymon\JWTAuth\Contracts\JWTSubject;
use Illuminate\Database\Eloquent\Relations\HasMany;

class User extends Authenticatable implements JWTSubject
{
    use HasFactory, Notifiable;

    protected $fillable = [
        'name',
        'email',
        'password',
        'role',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password' => 'hashed',
        ];
    }

    /** Kapcsolat: egy felhasználóhoz több ticket tartozhat. */
    public function tickets(): HasMany
    {
        return $this->hasMany(Ticket::class);
    }

    /** Kapcsolat: egy felhasználó több ticket válaszhoz tartozhat. */
    public function replies(): HasMany
    {
        return $this->hasMany(TicketReply::class);
    }

    // JWTSubject implementáció
    public function getJWTIdentifier()
    {
        return $this->getKey();
    }

    public function getJWTCustomClaims(): array
    {
        return [];
    }
}
Ticket model (app/Models/Ticket.php)
php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Ticket extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'user_id',
        'subject',
        'description',
        'priority',
        'status',
    ];

    /** Kapcsolat: egy ticket egy felhasználóhoz tartozik. */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    /** Kapcsolat: egy tickethez több válasz tartozhat. */
    public function replies(): HasMany
    {
        return $this->hasMany(TicketReply::class);
    }
}
TicketReply model (app/Models/TicketReply.php)
php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\SoftDeletes;

class TicketReply extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'ticket_id',
        'user_id',
        'message',
    ];

    /** Kapcsolat: egy válasz egy tickethez tartozik. */
    public function ticket(): BelongsTo
    {
        return $this->belongsTo(Ticket::class);
    }

    /** Kapcsolat: egy válasz egy felhasználóhoz tartozik. */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
API végpontok
A Content-Type és az Accept header kulcsok mindig application/json formátumúak legyenek.
Érvénytelen vagy hiányzó token esetén a backendnek 401 Unauthorized választ kell visszaadnia:

json
{
  "message": "Unauthenticated."
}
Nem védett végpontok
GET /ping – API teszteléshez

POST /register – Regisztrációhoz

POST /login – Bejelentkezéshez (JWT token generálása)

Védett végpontok (Bearer Token szükséges)
POST /logout – Kijelentkezés

GET /me – Saját felhasználói adatok lekérése

GET /tickets – Összes ticket listázása

POST /tickets – Új ticket létrehozása

GET /tickets/{id} – Egy ticket megtekintése a válaszokkal együtt

PUT/PATCH /tickets/{id} – Ticket módosítása (pl. státusz, prioritás)

DELETE /tickets/{id} – Ticket törlése (Soft Delete)

POST /tickets/{id}/replies – Új válasz hozzáadása egy tickethez

(Opció: GET /tickets/{id}/replies – adott ticket válaszainak listázása – ha szükséges)

Soft Delete funkciók
A rendszer Soft Delete megközelítést használ a ticketek és ticket válaszok törlésekor, hasonlóan az orders/payments példához.

A rekord fizikailag megmarad az adatbázisban

A deleted_at mező kitöltésre kerül az aktuális időbélyeggel

A lekérdezések alapértelmezetten nem tartalmazzák a törölt rekordokat

A törölt rekordok később visszaállíthatók (restore()), ha erre építünk külön admin funkciókat

Előnyök:

Adatvédelem: Véletlen törlés esetén az adatok visszaállíthatók

Audit trail: Nyomon követhető, mikor és milyen ticket/válasz került törlésre

Compliance: Bizonyos szabályozások megkövetelik az auditálhatóságot

Hibák
400 Bad Request: Hibás formátumú kérés, pl. rossz JSON.

401 Unauthorized: Nincs vagy érvénytelen Bearer token.

404 Not Found: A kért erőforrás nem található (pl. ticket vagy reply).

422 Unprocessable Entity: Validációs hiba (hiányzó vagy hibás mezők).

Felhasználókezelés
POST /register
Új felhasználó regisztrációja. Az új felhasználók regisztráció után külön be kell jelentkezzenek token megszerzéséhez (JWT). A válasz hasonló felépítésű, mint a payment példában a regisztráció.

Kérés törzse:

json
{
  "name": "Test User",
  "email": "test@example.com",
  "password": "password123",
  "password_confirmation": "password123"
}
Válasz (sikeres regisztráció – 201 Created):

json
{
  "message": "Registration successful",
  "user": {
    "id": 11,
    "name": "Test User",
    "email": "test@example.com",
    "role": "user",
    "created_at": "2026-01-04T10:30:00.000000Z",
    "updated_at": "2026-01-04T10:30:00.000000Z"
  }
}
Válasz (ha az e-mail cím már foglalt – 422 Unprocessable Entity):

json
{
  "message": "The email has already been taken.",
  "errors": {
    "email": [
      "The email has already been taken."
    ]
  }
}
POST /login
Bejelentkezés e-mail címmel és jelszóval, JWT Bearer token megszerzése.

Kérés törzse:

json
{
  "email": "admin@example.com",
  "password": "Admin_Secret_Pw2026!"
}
Válasz (sikeres bejelentkezés – 200 OK):

json
{
  "message": "Login successful",
  "user": {
    "id": 1,
    "name": "Admin",
    "email": "admin@example.com",
    "role": "admin",
    "created_at": "2026-01-04T10:30:00.000000Z",
    "updated_at": "2026-01-04T10:30:00.000000Z"
  },
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9....",
  "token_type": "Bearer",
  "expires_in": 3600
}
Válasz (sikertelen bejelentkezés – 422 Unprocessable Entity):

json
{
  "message": "The provided credentials are incorrect.",
  "errors": {
    "email": [
      "The provided credentials are incorrect."
    ]
  }
}
A további védett végpontoknál a kérés headerjében meg kell adni a tokent:

http
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9....
POST /logout
A jelenlegi autentikált felhasználó kijelentkeztetése, a token érvénytelenítése.

Válasz (sikeres kijelentkezés – 200 OK):

json
{
  "message": "Logout successful"
}
GET /me
Saját felhasználói profil lekérése.

Válasz (200 OK):

json
{
  "id": 1,
  "name": "Admin",
  "email": "admin@example.com",
  "role": "admin",
  "created_at": "2026-01-04T10:30:00.000000Z",
  "updated_at": "2026-01-04T10:30:00.000000Z"
}
Ticket kezelés
POST /tickets
Új ticket létrehozása bejelentkezett felhasználó nevében. A user_id a backendben az auth alapján kerül beállításra.

Kérés törzse:

json
{
  "subject": "Nem tudok belépni a fiókomba",
  "description": "Próbálok belépni, de mindig hibát ír.",
  "priority": "high"
}
Válasz (sikeres létrehozás – 201 Created):

json
{
  "success": true,
  "message": "Ticket created successfully",
  "data": {
    "id": 1,
    "user_id": 5,
    "subject": "Nem tudok belépni a fiókomba",
    "description": "Próbálok belépni, de mindig hibát ír.",
    "priority": "high",
    "status": "open",
    "created_at": "2026-01-04T11:00:00.000000Z",
    "updated_at": "2026-01-04T11:00:00.000000Z",
    "user": {
      "id": 5,
      "name": "Teszt Felhasználó",
      "email": "user5@example.com"
    }
  }
}
Hibák:

422 Unprocessable Entity – érvénytelen vagy hiányzó mezők (pl. üres subject)

401 Unauthorized – ha a token érvénytelen vagy hiányzik

GET /tickets
Az összes ticket listázása. (Alap esetben minden ticket; jogosultsági logika kiterjeszthető, pl. user csak a sajátját látja, admin mindent.)

Válasz (200 OK):

json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "user_id": 5,
      "subject": "Nem tudok belépni a fiókomba",
      "description": "Próbálok belépni, de mindig hibát ír.",
      "priority": "high",
      "status": "open",
      "created_at": "2026-01-04T11:00:00.000000Z",
      "user": {
        "id": 5,
        "name": "Teszt Felhasználó",
        "email": "user5@example.com"
      }
    }
  ]
}
GET /tickets/{id}
Információk lekérése egy adott ticketről, válaszokkal együtt.

Válasz (200 OK):

json
{
  "success": true,
  "data": {
    "id": 1,
    "user_id": 5,
    "subject": "Nem tudok belépni a fiókomba",
    "description": "Próbálok belépni, de mindig hibát ír.",
    "priority": "high",
    "status": "in_progress",
    "created_at": "2026-01-04T11:00:00.000000Z",
    "updated_at": "2026-01-04T12:00:00.000000Z",
    "user": {
      "id": 5,
      "name": "Teszt Felhasználó"
    },
    "replies": [
      {
        "id": 1,
        "ticket_id": 1,
        "user_id": 1,
        "message": "Megnézem a problémát, kérlek várj.",
        "created_at": "2026-01-04T11:30:00.000000Z",
        "user": {
          "id": 1,
          "name": "Admin"
        }
      }
    ]
  }
}
Válasz (ha a ticket nem található – 404 Not Found):

json
{
  "success": false,
  "message": "Ticket not found"
}
PUT/PATCH /tickets/{id}
Ticket adatainak frissítése.

PUT: minden mező kötelező (subject, description, priority, status)

PATCH: csak a módosítani kívánt mezők.

Kérés törzse (PUT):

json
{
  "subject": "Nem tudok belépni",
  "description": "Frissítettem a böngészőt, de még mindig hiba.",
  "priority": "medium",
  "status": "in_progress"
}
Kérés törzse (PATCH):

json
{
  "status": "closed"
}
Válasz (sikeres frissítés – 200 OK):

json
{
  "success": true,
  "message": "Ticket updated successfully",
  "data": {
    "id": 1,
    "subject": "Nem tudok belépni",
    "priority": "medium",
    "status": "closed",
    "updated_at": "2026-01-04T13:00:00.000000Z"
  }
}
Hibák:

422 Unprocessable Entity – érvénytelen mezők

404 Not Found – ticket nem található

401 Unauthorized – token érvénytelen

DELETE /tickets/{id}
Egy ticket soft delete törlése. A rekord fizikailag megmarad az adatbázisban, csak a deleted_at mező kerül kitöltésre.

Válasz (sikeres törlés – 200 OK):

json
{
  "success": true,
  "message": "Ticket deleted successfully"
}
Válasz (ticket nem található – 404 Not Found):

json
{
  "success": false,
  "message": "Ticket not found"
}
POST /tickets/{id}/replies
Új válasz hozzáadása egy tickethez (user vagy admin).

Kérés törzse:

json
{
  "message": "Kérlek próbálj meg egy másik böngészőt."
}
Válasz (sikeres létrehozás – 201 Created):

json
{
  "success": true,
  "message": "Reply created successfully",
  "data": {
    "id": 10,
    "ticket_id": 1,
    "user_id": 1,
    "message": "Kérlek próbálj meg egy másik böngészőt.",
    "created_at": "2026-01-04T13:10:00.000000Z",
    "user": {
      "id": 1,
      "name": "Admin"
    }
  }
}
Összefoglaló táblázat
HTTP metódus	Útvonal	Jogosultság	Státuszkódok	Rövid leírás
GET	/ping	Nyilvános	200 OK	API tesztelés
POST	/register	Nyilvános	201, 422	Új felhasználó regisztrációja
POST	/login	Nyilvános	200, 422	Bejelentkezés, JWT token kiadása
POST	/logout	Hitelesített	200, 401	Kijelentkezés
GET	/me	Hitelesített	200, 401	Saját profil lekérése
GET	/tickets	Hitelesített	200, 401	Összes ticket listázása
POST	/tickets	Hitelesített	201, 422, 401	Új ticket létrehozása
GET	/tickets/{id}	Hitelesített	200, 404, 401	Ticket részletei válaszokkal együtt
PUT/PATCH	/tickets/{id}	Hitelesített	200, 422, 404, 401	Ticket frissítése
DELETE	/tickets/{id}	Hitelesített	200, 404, 401	Ticket törlése (Soft Delete)
POST	/tickets/{id}/replies	Hitelesített	201, 422, 404, 401	Új válasz egy tickethez
Factory-k és seederek
A felépítés a Payment példában látott Order/Payment factory-hez hasonló, de ticket domainre szabva.

UserFactory (database/factories/UserFactory.php)
php
<?php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<\App\Models\User>
 */
class UserFactory extends Factory
{
    public function definition(): array
    {
        return [
            'name' => fake('hu_HU')->name(),
            'email' => fake('hu_HU')->unique()->safeEmail(),
            'password' => 'password', // hashed cast miatt jó
            'role' => 'user',
        ];
    }
}
TicketFactory (database/factories/TicketFactory.php)
php
<?php

namespace Database\Factories;

use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<\App\Models\Ticket>
 */
class TicketFactory extends Factory
{
    public function definition(): array
    {
        return [
            'user_id' => User::factory(),
            'subject' => fake('hu_HU')->sentence(),
            'description' => fake('hu_HU')->paragraph(),
            'priority' => fake()->randomElement(['low', 'medium', 'high']),
            'status' => fake()->randomElement(['open', 'in_progress', 'closed']),
        ];
    }
}
TicketReplyFactory (database/factories/TicketReplyFactory.php)
php
<?php

namespace Database\Factories;

use App\Models\User;
use App\Models\Ticket;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<\App\Models\TicketReply>
 */
class TicketReplyFactory extends Factory
{
    public function definition(): array
    {
        return [
            'ticket_id' => Ticket::factory(),
            'user_id' => User::factory(),
            'message' => fake('hu_HU')->paragraph(),
        ];
    }
}
DatabaseSeeder (database/seeders/DatabaseSeeder.php)
php
<?php

namespace Database\Seeders;

use App\Models\User;
use App\Models\Ticket;
use App\Models\TicketReply;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        // Admin felhasználó
        $admin = User::factory()->create([
            'name' => 'Admin',
            'email' => 'admin@example.com',
            'password' => bcrypt('Admin_Secret_Pw2026!'),
            'role' => 'admin',
        ]);

        // 10 user felhasználó magyar adatokkal
        User::factory(10)->create()->each(function ($user) use ($admin) {
            // Minden userhez 1-5 ticket
            Ticket::factory(rand(1, 5))->create([
                'user_id' => $user->id,
            ])->each(function ($ticket) use ($user, $admin) {
                // Minden tickethez 0-5 válasz, véletlenszerűen user vagy admin
                TicketReply::factory(rand(0, 5))->create([
                    'ticket_id' => $ticket->id,
                    'user_id' => fake()->boolean(40) ? $admin->id : $user->id,
                ]);
            });
        });
    }
}
Létrehozott adatok:

1 admin felhasználó:  
email: admin@example.com, jelszó: Admin_Secret_Pw2026!

10 user felhasználó: magyar nevekkel (APP_FAKER_LOCALE=hu_HU)

10–50 ticket: minden felhasználóhoz 1–5 ticket

0–250 válasz: minden tickethez 0–5 válasz

Seeder futtatása:

bash
php artisan db:seed
Adatbázis frissítése és újra feltöltése:

bash
php artisan migrate:fresh --seed
Adatbázis diagram (szöveges leírás)
Hasonlóan a payment példában használt erDiagram-hoz, itt a relációk:

USERS ||--o{ TICKETS : owns

TICKETS ||--o{ TICKET_REPLIES : has

USERS ||--o{ TICKET_REPLIES : writes

Megvalósítási útmutató
I. Modul – Környezet beállítása
1. Projekt létrehozása és függőségek telepítése

bash
composer create-project laravel/laravel SupportTicketsJWT
cd SupportTicketsJWT

composer require tymon/jwt-auth
php artisan vendor:publish --provider="Tymon\JWTAuth\Providers\LaravelServiceProvider"
php artisan jwt:secret
2. .env fájl beállítása

env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=supportPlatform
DB_USERNAME=root
DB_PASSWORD=

APP_TIMEZONE=Europe/Budapest
APP_FAKER_LOCALE=hu_HU
3. Auth guard beállítása (config/auth.php)

php
'guards' => [
    'api' => [
        'driver' => 'jwt',
        'provider' => 'users',
    ],
],
II. Modul – Migrációk
Users tábla bővítése role mezővel  
A meglévő users migrációban vagy új migrációban:

php
Schema::table('users', function (Blueprint $table) {
    $table->string('role')->default('user');
});
Tickets tábla

bash
php artisan make:migration create_tickets_table
php
Schema::create('tickets', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->string('subject');
    $table->text('description');
    $table->enum('priority', ['low', 'medium', 'high'])->default('low');
    $table->enum('status', ['open', 'in_progress', 'closed'])->default('open');
    $table->timestamps();
    $table->softDeletes();
});
Ticket_replies tábla

bash
php artisan make:migration create_ticket_replies_table
php
Schema::create('ticket_replies', function (Blueprint $table) {
    $table->id();
    $table->foreignId('ticket_id')->constrained()->onDelete('cascade');
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->text('message');
    $table->timestamps();
    $table->softDeletes();
});
Migrációk futtatása

bash
php artisan migrate
III. Modul – Controller-ek és route-ok
AuthController (app/Http/Controllers/AuthController.php)

php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Validator;

class AuthController extends Controller
{
    public function register(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'name'     => 'required|string|max:255',
            'email'    => 'required|string|email|max:255|unique:users',
            'password' => 'required|string|min:8|confirmed',
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors(), 422);
        }

        $user = User::create([
            'name'     => $request->name,
            'email'    => $request->email,
            'password' => Hash::make($request->password),
            'role'     => 'user',
        ]);

        return response()->json([
            'message' => 'Registration successful',
            'user'    => $user,
        ], 201);
    }

    public function login(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'email'    => 'required|email',
            'password' => 'required',
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors(), 422);
        }

        $credentials = $request->only('email', 'password');

        if (!$token = auth('api')->attempt($credentials)) {
            return response()->json([
                'message' => 'The provided credentials are incorrect.',
                'errors'  => ['email' => ['The provided credentials are incorrect.']],
            ], 422);
        }

        return response()->json([
            'message'      => 'Login successful',
            'user'         => auth('api')->user(),
            'access_token' => $token,
            'token_type'   => 'Bearer',
            'expires_in'   => auth('api')->factory()->getTTL() * 60,
        ]);
    }

    public function me()
    {
        return response()->json(auth('api')->user());
    }

    public function logout()
    {
        auth('api')->logout();

        return response()->json(['message' => 'Logout successful']);
    }
}
TicketController (app/Http/Controllers/TicketController.php)

php
<?php

namespace App\Http\Controllers;

use App\Models\Ticket;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Validator;

class TicketController extends Controller
{
    public function index()
    {
        $tickets = Ticket::with('user')->get();

        return response()->json([
            'success' => true,
            'data'    => $tickets,
        ]);
    }

    public function store(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'subject'     => 'required|string|max:255',
            'description' => 'required|string',
            'priority'    => 'required|in:low,medium,high',
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors(), 422);
        }

        $ticket = Ticket::create([
            'user_id'     => auth('api')->id(),
            'subject'     => $request->subject,
            'description' => $request->description,
            'priority'    => $request->priority,
            'status'      => 'open',
        ]);

        $ticket->load('user');

        return response()->json([
            'success' => true,
            'message' => 'Ticket created successfully',
            'data'    => $ticket,
        ], 201);
    }

    public function show(string $id)
    {
        $ticket = Ticket::with(['user', 'replies.user'])->find($id);

        if (!$ticket) {
            return response()->json([
                'success' => false,
                'message' => 'Ticket not found',
            ], 404);
        }

        return response()->json([
            'success' => true,
            'data'    => $ticket,
        ]);
    }

    public function update(Request $request, string $id)
    {
        $ticket = Ticket::find($id);

        if (!$ticket) {
            return response()->json([
                'success' => false,
                'message' => 'Ticket not found',
            ], 404);
        }

        $rules = $request->isMethod('put')
            ? [
                'subject'     => 'required|string|max:255',
                'description' => 'required|string',
                'priority'    => 'required|in:low,medium,high',
                'status'      => 'required|in:open,in_progress,closed',
            ]
            : [
                'subject'     => 'sometimes|string|max:255',
                'description' => 'sometimes|string',
                'priority'    => 'sometimes|in:low,medium,high',
                'status'      => 'sometimes|in:open,in_progress,closed',
            ];

        $validator = Validator::make($request->all(), $rules);

        if ($validator->fails()) {
            return response()->json($validator->errors(), 422);
        }

        $ticket->update($request->all());
        $ticket->load('user');

        return response()->json([
            'success' => true,
            'message' => 'Ticket updated successfully',
            'data'    => $ticket,
        ]);
    }

    public function destroy(string $id)
    {
        $ticket = Ticket::find($id);

        if (!$ticket) {
            return response()->json([
                'success' => false,
                'message' => 'Ticket not found',
            ], 404);
        }

        $ticket->delete();

        return response()->json([
            'success' => true,
            'message' => 'Ticket deleted successfully',
        ]);
    }
}
TicketReplyController (app/Http/Controllers/TicketReplyController.php)

php
<?php

namespace App\Http\Controllers;

use App\Models\Ticket;
use App\Models\TicketReply;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Validator;

class TicketReplyController extends Controller
{
    public function store(Request $request, string $ticketId)
    {
        $ticket = Ticket::find($ticketId);

        if (!$ticket) {
            return response()->json([
                'success' => false,
                'message' => 'Ticket not found',
            ], 404);
        }

        $validator = Validator::make($request->all(), [
            'message' => 'required|string',
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors(), 422);
        }

        $reply = TicketReply::create([
            'ticket_id' => $ticket->id,
            'user_id'   => auth('api')->id(),
            'message'   => $request->message,
        ]);

        $reply->load('user');

        return response()->json([
            'success' => true,
            'message' => 'Reply created successfully',
            'data'    => $reply,
        ], 201);
    }
}
Route-ok beállítása (routes/api.php)

php
<?php

use App\Http\Controllers\AuthController;
use App\Http\Controllers\TicketController;
use App\Http\Controllers\TicketReplyController;
use Illuminate\Support\Facades\Route;

// Ping endpoint - API teszteléshez
Route::get('/ping', function () {
    return response()->json(['message' => 'pong']);
});

// Publikus végpontok
Route::post('/register', [AuthController::class, 'register']);
Route::post('/login',    [AuthController::class, 'login']);

// Védett végpontok (JWT Bearer szükséges)
Route::middleware('auth:api')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout']);
    Route::get('/me',      [AuthController::class, 'me']);

    // Ticket CRUD
    Route::get('/tickets',          [TicketController::class, 'index']);
    Route::post('/tickets',         [TicketController::class, 'store']);
    Route::get('/tickets/{id}',     [TicketController::class, 'show']);
    Route::put('/tickets/{id}',     [TicketController::class, 'update']);
    Route::patch('/tickets/{id}',   [TicketController::class, 'update']);
    Route::delete('/tickets/{id}',  [TicketController::class, 'destroy']);

    // Ticket reply
    Route::post('/tickets/{id}/replies', [TicketReplyController::class, 'store']);
});
Tesztelés
A felépítés a Payment példában látható AuthTest és PaymentTest mintáját követi, de ticket domainre alkalmazva.

AuthTest (tests/Feature/AuthTest.php – példa)
php
<?php

namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class AuthTest extends TestCase
{
    use RefreshDatabase;

    public function test_ping_endpoint_returns_pong(): void
    {
        $response = $this->getJson('/api/ping');

        $response->assertStatus(200)
            ->assertJson(['message' => 'pong']);
    }

    public function test_user_can_register_with_valid_data(): void
    {
        $response = $this->postJson('/api/register', [
            'name'                  => 'Test User',
            'email'                 => 'test@example.com',
            'password'              => 'password123',
            'password_confirmation' => 'password123',
        ]);

        $response->assertStatus(201)
            ->assertJsonStructure(['message', 'user']);

        $this->assertDatabaseHas('users', ['email' => 'test@example.com']);
    }

    public function test_user_cannot_register_with_existing_email(): void
    {
        User::factory()->create(['email' => 'existing@example.com']);

        $response = $this->postJson('/api/register', [
            'name'                  => 'Test User',
            'email'                 => 'existing@example.com',
            'password'              => 'password123',
            'password_confirmation' => 'password123',
        ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['email']);
    }

    public function test_user_can_login_with_valid_credentials(): void
    {
        $user = User::factory()->create([
            'password' => bcrypt('password123'),
        ]);

        $response = $this->postJson('/api/login', [
            'email'    => $user->email,
            'password' => 'password123',
        ]);

        $response->assertStatus(200)
            ->assertJsonStructure(['message', 'user', 'access_token', 'token_type']);
    }

    public function test_authenticated_user_can_logout(): void
    {
        $user = User::factory()->create();
        $token = auth('api')->login($user);

        $response = $this->withHeaders([
            'Authorization' => 'Bearer '.$token,
        ])->postJson('/api/logout');

        $response->assertStatus(200)
            ->assertJson(['message' => 'Logout successful']);
    }
}
TicketTest (tests/Feature/TicketTest.php – példa)
php
<?php

namespace Tests\Feature;

use App\Models\Ticket;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class TicketTest extends TestCase
{
    use RefreshDatabase;

    private function authenticatedUser(): array
    {
        $user = User::factory()->create();
        $token = auth('api')->login($user);

        return ['user' => $user, 'token' => $token];
    }

    public function test_can_create_ticket_with_valid_data(): void
    {
        $auth = $this->authenticatedUser();

        $response = $this->withHeaders([
            'Authorization' => 'Bearer '.$auth['token'],
        ])->postJson('/api/tickets', [
            'subject'     => 'Teszt ticket',
            'description' => 'Ez egy teszt ticket leírása',
            'priority'    => 'high',
        ]);

        $response->assertStatus(201)
            ->assertJsonStructure(['success', 'message', 'data']);

        $this->assertDatabaseHas('tickets', [
            'subject' => 'Teszt ticket',
        ]);
    }

    public function test_cannot_create_ticket_without_authentication(): void
    {
        $response = $this->postJson('/api/tickets', [
            'subject'     => 'Teszt ticket',
            'description' => 'Ez egy teszt',
            'priority'    => 'high',
        ]);

        $response->assertStatus(401);
    }

    public function test_can_list_tickets_with_authentication(): void
    {
        $auth = $this->authenticatedUser();
        Ticket::factory()->count(3)->create(['user_id' => $auth['user']->id]);

        $response = $this->withHeaders([
            'Authorization' => 'Bearer '.$auth['token'],
        ])->getJson('/api/tickets');

        $response->assertStatus(200)
            ->assertJsonStructure(['success', 'data']);
    }

    public function test_can_show_single_ticket(): void
    {
        $auth = $this->authenticatedUser();
        $ticket = Ticket::factory()->create(['user_id' => $auth['user']->id]);

        $response = $this->withHeaders([
            'Authorization' => 'Bearer '.$auth['token'],
        ])->getJson('/api/tickets/'.$ticket->id);

        $response->assertStatus(200)
            ->assertJson(['success' => true]);
    }

    public function test_can_delete_ticket_soft_delete(): void
    {
        $auth = $this->authenticatedUser();
        $ticket = Ticket::factory()->create(['user_id' => $auth['user']->id]);

        $response = $this->withHeaders([
            'Authorization' => 'Bearer '.$auth['token'],
        ])->deleteJson('/api/tickets/'.$ticket->id);

        $response->assertStatus(200)
            ->assertJson(['success' => true, 'message' => 'Ticket deleted successfully']);

        $this->assertSoftDeleted('tickets', ['id' => $ticket->id]);
    }
}
Tesztek futtatása:

bash
php artisan test
Várt eredmény: minden Auth és Ticket Feature teszt PASS, hasonló kimenettel, mint a payment projekt esetén.

Telepítési útmutató
1. Adatbázis konfigurálása:

Hozz létre egy supportPlatform nevű adatbázist

Állítsd be a .env fájlban az adatbázis kapcsolatot

2. Függőségek telepítése és migrációk + seed:

bash
composer install
php artisan migrate
php artisan db:seed
3. Fejlesztői szerver indítása:

bash
php artisan serve
4. API elérése:

Base URL: http://127.0.0.1:8000/api

Teszt admin felhasználó:
admin@example.com / Admin_Secret_Pw2026!