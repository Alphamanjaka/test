# 🏗️ Architecture Backend — CineTrack
> Semaine 1 · Responsable : Alpha  
> Stack : Express + TypeScript · SQLite (`better-sqlite3`) · MongoDB (`mongoose`)

---

## 1. Répartition des bases de données

| Base | Responsabilité | Justification |
|------|---------------|---------------|
| **SQLite** | Users, Auth (sessions/tokens) | Données relationnelles simples, transactions ACID, pas de scaling horizontal nécessaire |
| **MongoDB** | Library entries, Thematic Lists | Documents flexibles (metadata TMDB variable), requêtes texte, évolution du schéma facile |

---

## 2. Structure de dossiers — Architecture Feature-Based

```
server/
├── src/
│   ├── features/
│   │   │
│   │   ├── auth/                          🔐 Authentication & Authorization
│   │   │   ├── auth.routes.ts             → POST /api/auth/register, /login, /logout
│   │   │   ├── auth.service.ts            → bcrypt, JWT signing, refresh tokens
│   │   │   ├── auth.repository.ts         → CRUD users (SQLite)
│   │   │   ├── auth.middleware.ts         → verifyJWT → req.user
│   │   │   ├── auth.schema.ts             → zod validation: RegisterDTO, LoginDTO
│   │   │   ├── auth.types.ts              → UserRow, PublicUser, JwtPayload
│   │   │   └── models/
│   │   │       ├── User.ts                → SQLite schema types & helpers
│   │   │       └── migrations/
│   │   │           ├── runner.ts          → executes migrations in order
│   │   │           └── 001_create_users.ts
│   │   │
│   │   ├── library/                       📚 User's Library Management
│   │   │   ├── library.routes.ts          → GET /api/library (filters, pagination)
│   │   │   │                                 POST /api/library (add entry)
│   │   │   │                                 PUT /api/library/:id (update status, rating)
│   │   │   │                                 DELETE /api/library/:id
│   │   │   ├── library.service.ts         → filters, validation, aggregations
│   │   │   ├── library.repository.ts      → CRUD MongoDB + aggregations for stats
│   │   │   ├── library.schema.ts          → zod: AddEntryDTO, UpdateEntryDTO
│   │   │   ├── library.types.ts           → ILibraryEntry, WatchStatus, LibraryFilters
│   │   │   └── models/
│   │   │       └── LibraryEntry.ts        → MongoDB schema + interface
│   │   │
│   │   ├── stats/                         📊 User Statistics & Analytics
│   │   │   ├── stats.routes.ts            → GET /api/stats/:userId (month breakdown, watch time)
│   │   │   ├── stats.service.ts           → orchestration (calls library aggregations)
│   │   │   ├── stats.types.ts             → StatsResult, MonthStat, GenreStat
│   │   │   └── [depends on library]       ← reads library entries via aggregations
│   │   │
│   │   ├── lists/                         ✨ Thematic Lists & Drag-Drop
│   │   │   ├── lists.routes.ts            → GET /api/lists, POST, PUT (reorder), DELETE
│   │   │   ├── lists.service.ts           → list operations, item reordering
│   │   │   ├── lists.repository.ts        → CRUD MongoDB + reorder (drag & drop)
│   │   │   ├── lists.schema.ts            → zod: CreateListDTO, ReorderItemsDTO
│   │   │   ├── lists.types.ts             → IThematicList, ListItem
│   │   │   └── models/
│   │   │       └── ThematicList.ts        → MongoDB schema + interface
│   │   │
│   │   └── shared/                        🔧 Shared Resources (cross-feature)
│   │       ├── config/
│   │       │   ├── sqlite.ts              → singleton DB connection + pragmas
│   │       │   ├── mongo.ts               → mongoose.connect()
│   │       │   └── env.ts                 → zod validation of .env
│   │       │
│   │       ├── middleware/
│   │       │   ├── errorHandler.ts        → global Express error handler
│   │       │   └── validate.ts            → zod middleware wrapper
│   │       │
│   │       ├── services/
│   │       │   └── http.ts                → shared HTTP utilities (if needed)
│   │       │
│   │       ├── types.ts                   → global: AuthRequest, ApiResponse, JwtPayload
│   │       └── utils.ts                   → helpers (uuid generation, timestamps, etc.)
│   │
│   ├── router/
│   │   └── index.ts                       → mounts all feature routes under /api
│   │
│   ├── seed/
│   │   └── migrate-seed.ts                → one-shot: seed.ts → SQLite users
│   │
│   └── index.ts                           → bootstrap Express + error handling
│
├── .env.example
├── .gitignore
├── package.json
└── tsconfig.json
```

---

## 3. Feature Dependencies — Golden Rule

```
Allowed Imports:
  ✅ features/* → shared/           (all features use shared utilities, config, types)
  ✅ features/stats → features/library  (stats orchestrates library aggregations)
  ✅ features/library → features/auth   (auth middleware protects library routes)

Forbidden:
  ❌ features/X → features/Y  (except stats→library)   Cross-feature coupling forbidden
  ❌ features/X → seed/       Features don't import seed
```

---

## 4. Feature: Auth — SQLite Models & Implementation

### `src/features/auth/models/User.ts`
```typescript
// Type qui reflète exactement la table SQLite
export interface UserRow {
  id: string;
  username: string;
  email: string;
  password: string;       // hash bcrypt, jamais exposé
  avatar_url: string | null;
  bio: string;
  is_private: 0 | 1;
  created_at: string;
  updated_at: string;
}

// DTO sans le mot de passe — ce qu'on renvoie au client
export type PublicUser = Omit<UserRow, 'password'>;
```

### `src/features/auth/models/migrations/001_create_users.ts`
```typescript
import db from '../../../shared/config/sqlite';

export function up(): void {
  db.exec(`
    CREATE TABLE IF NOT EXISTS users (
      id          TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
      username    TEXT UNIQUE NOT NULL,
      email       TEXT UNIQUE NOT NULL,
      password    TEXT NOT NULL,          -- bcrypt hash
      avatar_url  TEXT,
      bio         TEXT DEFAULT '',
      is_private  INTEGER DEFAULT 0,      -- 0 = public, 1 = privé (SQLite pas de BOOLEAN)
      created_at  TEXT DEFAULT (datetime('now')),
      updated_at  TEXT DEFAULT (datetime('now'))
    );

    CREATE TABLE IF NOT EXISTS refresh_tokens (
      id          TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
      user_id     TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
      token       TEXT UNIQUE NOT NULL,
      expires_at  TEXT NOT NULL,
      created_at  TEXT DEFAULT (datetime('now'))
    );

    CREATE INDEX IF NOT EXISTS idx_users_username ON users(username);
    CREATE INDEX IF NOT EXISTS idx_users_email    ON users(email);
    CREATE INDEX IF NOT EXISTS idx_tokens_user_id ON refresh_tokens(user_id);
  `);
}

export function down(): void {
  db.exec(`
    DROP TABLE IF EXISTS refresh_tokens;
    DROP TABLE IF EXISTS users;
  `);
}
```

### `src/features/auth/models/migrations/runner.ts`
```typescript
import db from '../../../shared/config/sqlite';
import { up as migration001 } from './001_create_users';

// Table interne pour suivre les migrations appliquées
db.exec(`
  CREATE TABLE IF NOT EXISTS _migrations (
    name       TEXT PRIMARY KEY,
    applied_at TEXT DEFAULT (datetime('now'))
  );
`);

const migrations: { name: string; up: () => void }[] = [
  { name: '001_create_users', up: migration001 },
];

export function runMigrations(): void {
  const applied = db
    .prepare('SELECT name FROM _migrations')
    .all()
    .map((r: any) => r.name);

  for (const migration of migrations) {
    if (!applied.includes(migration.name)) {
      console.log(`[migration] Applying ${migration.name}...`);
      migration.up();
      db.prepare('INSERT INTO _migrations (name) VALUES (?)').run(migration.name);
      console.log(`[migration] ✓ ${migration.name}`);
    }
  }
}
```

### `src/features/auth/auth.types.ts`
```typescript
import { PublicUser } from './models/User';

export interface RegisterDTO {
  username: string;
  email: string;
  password: string;
}

export interface LoginDTO {
  email: string;
  password: string;
}

export interface AuthResponse {
  user: PublicUser;
  access_token: string;
  refresh_token?: string;
}

export interface JwtPayload {
  sub: string;
  username: string;
  email: string;
  iat?: number;
  exp?: number;
}
```

### `src/features/auth/auth.schema.ts`
```typescript
import { z } from 'zod';

export const RegisterSchema = z.object({
  username: z.string().min(3).max(20),
  email: z.string().email(),
  password: z.string().min(8),
});

export const LoginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
});
```

### `src/features/auth/auth.repository.ts`
```typescript
import db from '../../shared/config/sqlite';
import { UserRow, PublicUser } from './models/User';

export const authRepository = {
  findByEmail(email: string): UserRow | undefined {
    return db.prepare('SELECT * FROM users WHERE email = ?').get(email) as UserRow | undefined;
  },

  findByUsername(username: string): UserRow | undefined {
    return db.prepare('SELECT * FROM users WHERE username = ?').get(username) as UserRow | undefined;
  },

  findById(id: string): UserRow | undefined {
    return db.prepare('SELECT * FROM users WHERE id = ?').get(id) as UserRow | undefined;
  },

  create(id: string, username: string, email: string, passwordHash: string): UserRow {
    db.prepare(`
      INSERT INTO users (id, username, email, password)
      VALUES (@id, @username, @email, @password)
    `).run({ id, username, email, password: passwordHash });
    return this.findById(id) as UserRow;
  },

  saveRefreshToken(userId: string, token: string, expiresAt: string): void {
    db.prepare(`
      INSERT INTO refresh_tokens (id, user_id, token, expires_at)
      VALUES (lower(hex(randomblob(16))), @user_id, @token, @expires_at)
    `).run({ user_id: userId, token, expires_at });
  },

  toPublic(user: UserRow): PublicUser {
    const { password, ...publicUser } = user;
    return publicUser;
  },
};
```

### `src/features/auth/auth.service.ts`
```typescript
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import { v4 as uuidv4 } from 'uuid';
import { authRepository } from './auth.repository';
import { RegisterDTO, LoginDTO, JwtPayload } from './auth.types';

export const authService = {
  async register(data: RegisterDTO) {
    const existing = authRepository.findByEmail(data.email);
    if (existing) throw new Error('Email already registered');

    const hash = bcrypt.hashSync(data.password, 10);
    const id = uuidv4();
    const user = authRepository.create(id, data.username, data.email, hash);
    return authRepository.toPublic(user);
  },

  async login(data: LoginDTO) {
    const user = authRepository.findByEmail(data.email);
    if (!user || !bcrypt.compareSync(data.password, user.password)) {
      throw new Error('Invalid credentials');
    }

    const payload: JwtPayload = {
      sub: user.id,
      username: user.username,
      email: user.email,
    };

    const accessToken = jwt.sign(payload, process.env.JWT_SECRET!, {
      expiresIn: process.env.JWT_EXPIRES_IN ?? '15m',
    });

    return { user: authRepository.toPublic(user), access_token: accessToken };
  },
};
```

### `src/features/auth/auth.middleware.ts`
```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { JwtPayload } from './auth.types';

export interface AuthRequest extends Request {
  user?: {
    id: string;
    username: string;
    email: string;
  };
}

export function verifyJWT(req: AuthRequest, res: Response, next: NextFunction): void {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) {
    res.status(401).json({ success: false, error: 'No token' });
    return;
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as JwtPayload;
    req.user = { id: decoded.sub, username: decoded.username, email: decoded.email };
    next();
  } catch (err) {
    res.status(401).json({ success: false, error: 'Invalid token' });
  }
}
```

### `src/features/auth/auth.routes.ts`
```typescript
import express, { Request, Response } from 'express';
import { authService } from './auth.service';
import { verifyJWT } from './auth.middleware';
import { LoginSchema, RegisterSchema } from './auth.schema';

const router = express.Router();

router.post('/register', async (req: Request, res: Response) => {
  try {
    const data = RegisterSchema.parse(req.body);
    const user = await authService.register(data);
    res.json({ success: true, data: user });
  } catch (err: any) {
    res.status(400).json({ success: false, error: err.message });
  }
});

router.post('/login', async (req: Request, res: Response) => {
  try {
    const data = LoginSchema.parse(req.body);
    const result = await authService.login(data);
    res.json({ success: true, data: result });
  } catch (err: any) {
    res.status(401).json({ success: false, error: err.message });
  }
});

export default router;
```

---

## 5. Feature: Library — MongoDB Models & Implementation

### `src/features/library/models/LibraryEntry.ts`
```typescript
import { Schema, model, Document } from 'mongoose';

export type WatchStatus = 'to_watch' | 'watching' | 'watched' | 'abandoned';

// Sous-document : données TMDB snapshotées à l'ajout
const TmdbSnapshotSchema = new Schema({
  title:        { type: String, required: true },
  poster_path:  { type: String, default: null },
  release_year: { type: Number, default: null },
  genres:       { type: [String], default: [] },
  runtime:      { type: Number, default: null }, // minutes — pour /stats temps de visionnage
  tmdb_rating:  { type: Number, default: null },
}, { _id: false });

const LibraryEntrySchema = new Schema({
  user_id:    { type: String, required: true, index: true },
  tmdb_id:    { type: Number, required: true },
  status:     { type: String, enum: ['to_watch', 'watching', 'watched', 'abandoned'], required: true },
  rating:     { type: Number, min: 0, max: 10, default: null },
  review:     { type: String, default: '' },
  is_public:  { type: Boolean, default: true },
  tmdb_data:  { type: TmdbSnapshotSchema, required: true },
}, { timestamps: true });

LibraryEntrySchema.index({ user_id: 1, tmdb_id: 1 }, { unique: true });
LibraryEntrySchema.index({ user_id: 1, status: 1, updatedAt: -1 });

export interface ILibraryEntry extends Document {
  user_id: string;
  tmdb_id: number;
  status: WatchStatus;
  rating: number | null;
  review: string;
  is_public: boolean;
  tmdb_data: { title: string; poster_path: string | null; release_year: number | null; genres: string[]; runtime: number | null; tmdb_rating: number | null };
  createdAt: Date;
  updatedAt: Date;
}

export const LibraryEntry = model<ILibraryEntry>('LibraryEntry', LibraryEntrySchema);
```

### `src/features/library/library.types.ts`
```typescript
import { WatchStatus } from './models/LibraryEntry';

export interface LibraryFilters {
  status?: WatchStatus;
  genre?: string;
  min_rating?: number;
  sort?: 'added' | 'rating';
}

export interface AddEntryDTO {
  tmdb_id: number;
  status: WatchStatus;
  tmdb_data: { title: string; poster_path?: string; genres?: string[]; runtime?: number };
}

export interface UpdateEntryDTO {
  status?: WatchStatus;
  rating?: number;
  review?: string;
}
```

### `src/features/library/library.schema.ts`
```typescript
import { z } from 'zod';

export const AddEntrySchema = z.object({
  tmdb_id: z.number(),
  status: z.enum(['to_watch', 'watching', 'watched', 'abandoned']),
  tmdb_data: z.object({
    title: z.string(),
    poster_path: z.string().optional(),
    genres: z.array(z.string()).optional(),
    runtime: z.number().optional(),
  }),
});

export const UpdateEntrySchema = z.object({
  status: z.enum(['to_watch', 'watching', 'watched', 'abandoned']).optional(),
  rating: z.number().min(0).max(10).optional(),
  review: z.string().optional(),
});
```

### `src/features/library/library.repository.ts`
```typescript
import { LibraryEntry, ILibraryEntry } from './models/LibraryEntry';
import { LibraryFilters } from './library.types';

export const libraryRepository = {
  async findByUser(userId: string, filters: LibraryFilters = {}) {
    const query: Record<string, unknown> = { user_id: userId };
    if (filters.status) query.status = filters.status;
    if (filters.genre) query['tmdb_data.genres'] = filters.genre;
    if (filters.min_rating) query.rating = { $gte: filters.min_rating };

    return LibraryEntry.find(query).sort({ createdAt: -1 });
  },

  async findOne(userId: string, tmdbId: number) {
    return LibraryEntry.findOne({ user_id: userId, tmdb_id: tmdbId });
  },

  async upsert(userId: string, tmdbId: number, data: Partial<ILibraryEntry>) {
    return LibraryEntry.findOneAndUpdate(
      { user_id: userId, tmdb_id: tmdbId },
      { $set: data },
      { upsert: true, new: true, runValidators: true }
    );
  },

  async delete(userId: string, tmdbId: number) {
    const result = await LibraryEntry.deleteOne({ user_id: userId, tmdb_id: tmdbId });
    return result.deletedCount === 1;
  },

  async watchedByMonth(userId: string) {
    return LibraryEntry.aggregate([
      { $match: { user_id: userId, status: 'watched' } },
      { $group: { _id: { $dateToString: { format: '%Y-%m', date: '$updatedAt' } }, count: { $sum: 1 } } },
      { $sort: { _id: 1 } },
    ]);
  },

  async totalWatchTime(userId: string) {
    const result = await LibraryEntry.aggregate([
      { $match: { user_id: userId, status: 'watched', 'tmdb_data.runtime': { $ne: null } } },
      { $group: { _id: null, total: { $sum: '$tmdb_data.runtime' } } },
    ]);
    return result[0]?.total ?? 0;
  },
};
```

### `src/features/library/library.service.ts`
```typescript
import { libraryRepository } from './library.repository';
import { AddEntryDTO, UpdateEntryDTO } from './library.types';

export const libraryService = {
  async getLibrary(userId: string, filters: any = {}) {
    return libraryRepository.findByUser(userId, filters);
  },

  async addEntry(userId: string, data: AddEntryDTO) {
    const existing = await libraryRepository.findOne(userId, data.tmdb_id);
    if (existing) throw new Error('Film déjà dans la bibliothèque');
    return libraryRepository.upsert(userId, data.tmdb_id, { ...data });
  },

  async updateEntry(userId: string, tmdbId: number, data: UpdateEntryDTO) {
    return libraryRepository.upsert(userId, tmdbId, data);
  },

  async deleteEntry(userId: string, tmdbId: number) {
    return libraryRepository.delete(userId, tmdbId);
  },
};
```

### `src/features/library/library.routes.ts`
```typescript
import express, { Response } from 'express';
import { verifyJWT, AuthRequest } from '../auth/auth.middleware';
import { libraryService } from './library.service';

const router = express.Router();

router.get('/', verifyJWT, async (req: AuthRequest, res: Response) => {
  try {
    const entries = await libraryService.getLibrary(req.user!.id, req.query);
    res.json({ success: true, data: entries });
  } catch (err: any) {
    res.status(500).json({ success: false, error: err.message });
  }
});

router.post('/', verifyJWT, async (req: AuthRequest, res: Response) => {
  try {
    const entry = await libraryService.addEntry(req.user!.id, req.body);
    res.json({ success: true, data: entry });
  } catch (err: any) {
    res.status(400).json({ success: false, error: err.message });
  }
});

router.put('/:tmdbId', verifyJWT, async (req: AuthRequest, res: Response) => {
  try {
    const entry = await libraryService.updateEntry(req.user!.id, parseInt(req.params.tmdbId), req.body);
    res.json({ success: true, data: entry });
  } catch (err: any) {
    res.status(400).json({ success: false, error: err.message });
  }
});

router.delete('/:tmdbId', verifyJWT, async (req: AuthRequest, res: Response) => {
  try {
    await libraryService.deleteEntry(req.user!.id, parseInt(req.params.tmdbId));
    res.json({ success: true });
  } catch (err: any) {
    res.status(400).json({ success: false, error: err.message });
  }
});

export default router;
```

---

## 6. Feature: Stats — Implementation

### `src/features/stats/stats.types.ts`
```typescript
export interface MonthStat {
  month: string;
  count: number;
}

export interface StatsResult {
  total_watched: number;
  total_watch_time: number; // minutes
  avg_rating: number;
  by_month: MonthStat[];
}
```

### `src/features/stats/stats.service.ts`
```typescript
import { libraryRepository } from '../library/library.repository';

export const statsService = {
  async getStats(userId: string) {
    const [watched, watchTime, byMonth] = await Promise.all([
      libraryRepository.findByUser(userId, { status: 'watched' }),
      libraryRepository.totalWatchTime(userId),
      libraryRepository.watchedByMonth(userId),
    ]);

    const avgRating = watched.length
      ? watched.reduce((sum, e) => sum + (e.rating || 0), 0) / watched.length
      : 0;

    return {
      total_watched: watched.length,
      total_watch_time: watchTime,
      avg_rating: Math.round(avgRating * 10) / 10,
      by_month: byMonth,
    };
  },
};
```

### `src/features/stats/stats.routes.ts`
```typescript
import express, { Response } from 'express';
import { verifyJWT, AuthRequest } from '../auth/auth.middleware';
import { statsService } from './stats.service';

const router = express.Router();

router.get('/', verifyJWT, async (req: AuthRequest, res: Response) => {
  try {
    const stats = await statsService.getStats(req.user!.id);
    res.json({ success: true, data: stats });
  } catch (err: any) {
    res.status(500).json({ success: false, error: err.message });
  }
});

export default router;
```

---

## 7. Feature: Lists — MongoDB Models & Implementation

### `src/features/lists/models/ThematicList.ts`
```typescript
import { Schema, model, Document } from 'mongoose';

const ThematicListSchema = new Schema({
  user_id:     { type: String, required: true, index: true },
  title:       { type: String, required: true },
  description: { type: String, default: '' },
  is_public:   { type: Boolean, default: true },
  items: [{
    tmdb_id:    { type: Number, required: true },
    title:      { type: String, required: true },
    poster_path:{ type: String, default: null },
    order:      { type: Number, required: true },
  }],
}, { timestamps: true });

export interface IThematicList extends Document {
  user_id: string;
  title: string;
  description: string;
  is_public: boolean;
  items: Array<{ tmdb_id: number; title: string; poster_path: string | null; order: number }>;
}

export const ThematicList = model<IThematicList>('ThematicList', ThematicListSchema);
```

### `src/features/lists/lists.types.ts`
```typescript
export interface ListItem {
  tmdb_id: number;
  title: string;
  poster_path: string | null;
  order: number;
}

export interface CreateListDTO {
  title: string;
  description?: string;
  is_public?: boolean;
}

export interface ReorderItemsDTO {
  items: Array<{ tmdb_id: number; order: number }>;
}
```

### `src/features/lists/lists.repository.ts`
```typescript
import { ThematicList, IThematicList } from './models/ThematicList';

export const listsRepository = {
  async findByUser(userId: string) {
    return ThematicList.find({ user_id: userId }).sort({ createdAt: -1 });
  },

  async findById(userId: string, listId: string) {
    return ThematicList.findOne({ _id: listId, user_id: userId });
  },

  async create(userId: string, data: { title: string; description?: string; is_public?: boolean }) {
    return ThematicList.create({ user_id: userId, ...data });
  },

  async update(userId: string, listId: string, data: Partial<IThematicList>) {
    return ThematicList.findOneAndUpdate({ _id: listId, user_id: userId }, data, { new: true });
  },

  async reorder(userId: string, listId: string, items: Array<{ tmdb_id: number; order: number }>) {
    return ThematicList.findOneAndUpdate(
      { _id: listId, user_id: userId },
      { items },
      { new: true }
    );
  },

  async delete(userId: string, listId: string) {
    const result = await ThematicList.deleteOne({ _id: listId, user_id: userId });
    return result.deletedCount === 1;
  },
};
```

### `src/features/lists/lists.service.ts`
```typescript
import { listsRepository } from './lists.repository';

export const listsService = {
  async getLists(userId: string) {
    return listsRepository.findByUser(userId);
  },

  async createList(userId: string, data: any) {
    return listsRepository.create(userId, data);
  },

  async reorderItems(userId: string, listId: string, items: any[]) {
    return listsRepository.reorder(userId, listId, items);
  },

  async deleteList(userId: string, listId: string) {
    return listsRepository.delete(userId, listId);
  },
};
```

### `src/features/lists/lists.routes.ts`
```typescript
import express, { Response } from 'express';
import { verifyJWT, AuthRequest } from '../auth/auth.middleware';
import { listsService } from './lists.service';

const router = express.Router();

router.get('/', verifyJWT, async (req: AuthRequest, res: Response) => {
  try {
    const lists = await listsService.getLists(req.user!.id);
    res.json({ success: true, data: lists });
  } catch (err: any) {
    res.status(500).json({ success: false, error: err.message });
  }
});

router.post('/', verifyJWT, async (req: AuthRequest, res: Response) => {
  try {
    const list = await listsService.createList(req.user!.id, req.body);
    res.json({ success: true, data: list });
  } catch (err: any) {
    res.status(400).json({ success: false, error: err.message });
  }
});

router.put('/:listId/reorder', verifyJWT, async (req: AuthRequest, res: Response) => {
  try {
    const list = await listsService.reorderItems(req.user!.id, req.params.listId, req.body.items);
    res.json({ success: true, data: list });
  } catch (err: any) {
    res.status(400).json({ success: false, error: err.message });
  }
});

router.delete('/:listId', verifyJWT, async (req: AuthRequest, res: Response) => {
  try {
    await listsService.deleteList(req.user!.id, req.params.listId);
    res.json({ success: true });
  } catch (err: any) {
    res.status(400).json({ success: false, error: err.message });
  }
});

export default router;
```

---

## 8. Shared Configuration & Types

### `src/shared/config/sqlite.ts`
```typescript
import Database from 'better-sqlite3';
import path from 'path';

const DB_PATH = process.env.SQLITE_PATH ?? path.join(__dirname, '../../data/cinetrack.db');

const db = new Database(DB_PATH);
db.pragma('journal_mode = WAL');
db.pragma('foreign_keys = ON');

export default db;
```

### `src/shared/config/mongo.ts`
```typescript
import mongoose from 'mongoose';

const MONGO_URI = process.env.MONGO_URI ?? 'mongodb://localhost:27017/cinetrack';

export async function connectMongo(): Promise<void> {
  try {
    await mongoose.connect(MONGO_URI);
    console.log('[mongo] Connected ✓');
  } catch (err) {
    console.error('[mongo] Connection failed:', err);
    process.exit(1);
  }
}
```

### `src/shared/types.ts`
```typescript
import { Request } from 'express';

export interface ApiResponse<T = unknown> {
  success: boolean;
  data?: T;
  error?: string;
}
```

### `src/shared/middleware/errorHandler.ts`
```typescript
import { Request, Response, NextFunction } from 'express';

export function errorHandler(err: Error, req: Request, res: Response, next: NextFunction): void {
  console.error('[error]', err);
  res.status(500).json({ success: false, error: err.message ?? 'Internal server error' });
}
```

---

## 9. Router & Entry Point

### `src/router/index.ts`
```typescript
import express from 'express';
import authRoutes from '../features/auth/auth.routes';
import libraryRoutes from '../features/library/library.routes';
import statsRoutes from '../features/stats/stats.routes';
import listsRoutes from '../features/lists/lists.routes';

const router = express.Router();

router.use('/auth', authRoutes);
router.use('/library', libraryRoutes);
router.use('/stats', statsRoutes);
router.use('/lists', listsRoutes);

export default router;
```

### `src/index.ts`
```typescript
import express from 'express';
import cors from 'cors';
import { runMigrations } from './features/auth/models/migrations/runner';
import { connectMongo } from './shared/config/mongo';
import { errorHandler } from './shared/middleware/errorHandler';
import router from './router';

async function bootstrap() {
  // 1. Migrations SQLite
  runMigrations();

  // 2. Connexion MongoDB
  await connectMongo();

  // 3. Express
  const app = express();
  app.use(cors({ origin: process.env.CLIENT_URL ?? 'http://localhost:5173', credentials: true }));
  app.use(express.json());

  app.use('/api', router);

  app.get('/health', (_req, res) => res.json({ status: 'ok' }));

  app.use(errorHandler);

  const PORT = process.env.PORT ?? 3000;
  app.listen(PORT, () => console.log(`[server] Running on http://localhost:${PORT}`));
}

bootstrap().catch(console.error);
```

---

## 10. Variables d'environnement

### `.env.example`
```env
# SQLite
SQLITE_PATH=./data/cinetrack.db

# MongoDB
MONGO_URI=mongodb://localhost:27017/cinetrack

# JWT
JWT_SECRET=change_me_in_production
JWT_EXPIRES_IN=15m
REFRESH_TOKEN_EXPIRES_IN=7d

# TMDB
VITE_TMDB_API_KEY=your_key_here

# Server
PORT=3000
CLIENT_URL=http://localhost:5173
```

### `.gitignore`
```
data/
*.db
*.db-wal
*.db-shm
.env
node_modules/
dist/
```

---

## 11. Dépendances à installer

```bash
cd server
npm install express cors better-sqlite3 mongoose bcryptjs jsonwebtoken uuid zod
npm install -D @types/express @types/better-sqlite3 @types/bcryptjs @types/jsonwebtoken @types/uuid @types/cors ts-node typescript
```

---

## 12. Seed Migration Strategy

### `src/seed/migrate-seed.ts`
```typescript
/**
 * One-shot script — execute ONCE after DB init
 * Usage: npx ts-node src/seed/migrate-seed.ts
 */
import bcrypt from 'bcryptjs';
import { v4 as uuidv4 } from 'uuid';
import db from '../shared/config/sqlite';
import { runMigrations } from '../features/auth/models/migrations/runner';

const SEED_USERS = [
  { username: 'alice',   email: 'alice@example.com',   password: 'password123' },
  { username: 'bob',     email: 'bob@example.com',     password: 'password123' },
  { username: 'charlie', email: 'charlie@example.com', password: 'password123' },
];

async function migrate() {
  runMigrations();

  const insert = db.prepare(`
    INSERT OR IGNORE INTO users (id, username, email, password)
    VALUES (@id, @username, @email, @password)
  `);

  const insertMany = db.transaction((users: typeof SEED_USERS) => {
    for (const user of users) {
      const hash = bcrypt.hashSync(user.password, 10);
      insert.run({ id: uuidv4(), username: user.username, email: user.email, password: hash });
      console.log(`[seed] Inserted: ${user.username}`);
    }
  });

  insertMany(SEED_USERS);
  console.log('[seed] ✓ Migration complete');
  process.exit(0);
}

migrate().catch(err => { console.error('[seed] Error:', err); process.exit(1); });
```

---

## 13. Checklist — Week 1

- [ ] Create folder structure (features/*, shared/*)
- [ ] Implement `shared/config/` (SQLite, MongoDB)
- [ ] Create `features/auth/models/` with User types & migrations
- [ ] Implement `features/auth/` (repository, service, middleware, routes)
- [ ] Run `migrate-seed.ts` to import mock users
- [ ] Create `features/library/models/LibraryEntry.ts` (MongoDB schema)
- [ ] Implement `features/library/` (repository, service, routes)
- [ ] Create `features/lists/models/ThematicList.ts` (MongoDB schema)
- [ ] Implement `features/lists/` (repository, service, routes)
- [ ] Implement `features/stats/` (service, routes)
- [ ] Test manually: register → login → JWT → protected routes
- [ ] Open PR: `feat/backend-db-setup`

---

## 14. API Routes Summary

```
POST   /api/auth/register          → Create account
POST   /api/auth/login             → Get JWT token
GET    /api/library                → List entries (with filters)
POST   /api/library                → Add entry
PUT    /api/library/:tmdbId        → Update status/rating
DELETE /api/library/:tmdbId        → Remove entry
GET    /api/stats                  → Get user statistics
GET    /api/lists                  → List all thematic lists
POST   /api/lists                  → Create list
PUT    /api/lists/:listId/reorder  → Reorder items (drag & drop)
DELETE /api/lists/:listId          → Delete list
```
