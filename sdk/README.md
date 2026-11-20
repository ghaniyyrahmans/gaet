# @ghaniyyrahmans/gaet

Official TypeScript and JavaScript client SDK for **Gaet**, a database backup and cloud sync CLI tool.

[![npm version](https://img.shields.io/npm/v/@ghaniyyrahmans/gaet.svg)](https://www.npmjs.com/package/@ghaniyyrahmans/gaet)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> **Note:** this package was previously published as `@ghanirahmans/gaet`. The name changed, the API did not. See [Migrating from `@ghanirahmans/gaet`](#migrating-from-ghanirahmansgaet).

## How It Works

This SDK is a lightweight TypeScript client wrapper for the `gaet serve` REST API (running by default at `http://127.0.0.1:6161`). Your web application calls SDK methods like `gaet.push()`, which send HTTP requests to the local Gaet daemon to perform database backups and syncs.

## Installation

```bash
npm install @ghaniyyrahmans/gaet
# or
pnpm add @ghaniyyrahmans/gaet
# or
yarn add @ghaniyyrahmans/gaet
# or
bun add @ghaniyyrahmans/gaet
```

## Prerequisites

Ensure the `gaet` service daemon is running locally or on your server:

```bash
# Start background server (default port: 6161)
gaet serve

# Or enable auto-start service (systemd, launchd, or Task Scheduler)
gaet serve --auto
```

## Quick Start

### Option A: Auto-Start Server from Code (Recommended for Node.js)

```typescript
import { gaet } from '@ghaniyyrahmans/gaet';

// Start daemon with inline environment configuration
await gaet.startServer({
  env: {
    GAET_LOCAL_DB_HOST: '127.0.0.1',
    GAET_LOCAL_DB_NAME: 'my_app_db',
    GAET_REMOTE_URL: 'postgresql://postgres:secret@aws.supabase.com:5432/postgres',
  },
});

// Trigger a backup to cloud
const pushResult = await gaet.push();
if (pushResult.ok) {
  console.log('Backup created:', pushResult.msg);
}

// Stop daemon process when app shuts down (optional)
await gaet.stopServer();
```

### Option B: Use Existing Background Server

Start `gaet serve` in a terminal or as an OS service (`gaet serve --auto`), then connect directly:

```typescript
import { gaet } from '@ghaniyyrahmans/gaet';

// Check connection status
const status = await gaet.status();
console.log('Local DB connected:', status.local_ok);

// Trigger a backup
await gaet.push();
```

## Code Examples

### Next.js App Router API Route (`app/api/backup/route.ts`)

```typescript
import { NextResponse } from 'next/server';
import { gaet } from '@ghaniyyrahmans/gaet';

export async function POST() {
  try {
    const result = await gaet.push();
    if (!result.ok) {
      return NextResponse.json({ error: result.msg }, { status: 500 });
    }
    return NextResponse.json({ success: true, msg: result.msg });
  } catch (error: any) {
    return NextResponse.json({ error: 'Gaet daemon unreachable: ' + error.message }, { status: 503 });
  }
}
```

### React Component

```tsx
import React, { useState } from 'react';
import { gaet } from '@ghaniyyrahmans/gaet';

export function BackupButton() {
  const [loading, setLoading] = useState(false);
  const [statusMsg, setStatusMsg] = useState('');

  const handleBackup = async () => {
    setLoading(true);
    try {
      const res = await gaet.push();
      setStatusMsg(res.ok ? `Backup created: ${res.msg}` : `Error: ${res.msg}`);
    } catch (err: any) {
      setStatusMsg('Daemon offline');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <button onClick={handleBackup} disabled={loading}>
        {loading ? 'Creating Backup...' : 'Push Backup'}
      </button>
      {statusMsg && <p>{statusMsg}</p>}
    </div>
  );
}
```

## Custom Client Options

```typescript
import { GaetClient } from '@ghaniyyrahmans/gaet';

const gaetClient = new GaetClient({
  baseUrl: 'http://127.0.0.1:6161',
  timeout: 120000, // 2 minutes timeout for large database operations
});

const checkResult = await gaetClient.check();
console.log('PostgreSQL tools ok:', checkResult.checks.tools.ok);
```

## API Reference

| Method | Return Type | Description |
| :--- | :--- | :--- |
| `gaet.startServer(options?)` | `Promise<{ ok: boolean; msg: string; pid?: number }>` | Auto-spawns `gaet serve` daemon process from Node.js code if not running. |
| `gaet.stopServer()` | `Promise<{ ok: boolean; msg: string }>` | Stops the spawned `gaet serve` process. |
| `gaet.status()` | `Promise<GaetStatusResponse>` | Returns local and cloud database connection status. |
| `gaet.push()` | `Promise<GaetPushResponse>` | Triggers database backup from local database to cloud remote. |
| `gaet.fetch()` | `Promise<GaetFetchResponse>` | Fetches cloud database state and restores it locally. |
| `gaet.restore(snapshotName?)` | `Promise<GaetFetchResponse>` | Restores the local database from a stored snapshot dump file. |
| `gaet.snapshots()` | `Promise<GaetSnapshotsResponse>` | Lists snapshot files stored in `~/.gaet/backups`. |
| `gaet.deleteSnapshot(snapshotName)` | `Promise<{ ok: boolean; msg: string }>` | Removes a specific snapshot dump file from disk. |
| `gaet.logs()` | `Promise<GaetLogsResponse>` | Reads audit log entries from `~/.gaet/backups/gaet.log` (and `cron.log`). |
| `gaet.check()` | `Promise<GaetCheckResponse>` | Runs system checks for `pg_dump`, `psql`, and permissions. |
| `gaet.doctor()` | `Promise<GaetDoctorResponse>` | Runs full doctor diagnostics on environment, config, and tools. |
| `gaet.diff()` | `Promise<GaetDiffResponse>` | Compares table counts between local and remote databases. |
| `gaet.detect()` | `Promise<GaetDetectResponse>` | Scans local system for active database socket and TCP instances. |
| `gaet.testRemote()` | `Promise<GaetRemoteTestResponse>` | Tests connectivity to the Cloud Remote database URL. |
| `gaet.getConfig()` | `Promise<{ config: Record<string, string>; masked_config: Record<string, string>; env_file: string }>` | Reads environment configuration variables from `~/.gaet/.env`. |
| `gaet.setConfig(config)` | `Promise<{ ok: boolean; msg: string }>` | Saves environment configuration variables to `~/.gaet/.env`. |
| `gaet.export()` | `Promise<GaetExportResponse>` | Exports configuration as shell environment statements. |

## TypeScript Types

```typescript
import type {
  GaetStatusResponse,
  GaetConfigResponse,
  GaetPushResponse,
  GaetFetchResponse,
  GaetSnapshotsResponse,
  GaetSnapshotItem,
  GaetCheckResponse,

  GaetLogsResponse,
  GaetLogEntry,
} from '@ghaniyyrahmans/gaet';
```

## Migrating from `@ghanirahmans/gaet`

The SDK API is unchanged. Only the package name moved.

```bash
npm uninstall @ghanirahmans/gaet
npm install @ghaniyyrahmans/gaet
```

```diff
- import { gaet } from '@ghanirahmans/gaet';
+ import { gaet } from '@ghaniyyrahmans/gaet';
```

`@ghanirahmans/gaet` is deprecated and receives no further updates.

## License

MIT © [Ghaniyyir Rahman Sudarsono](https://github.com/ghaniyyrahmans)
