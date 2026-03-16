const { neon } = require('@neondatabase/serverless');

const sql = neon(process.env.NETLIFY_DATABASE_URL);

async function ensureSchema() {
  await sql`
    CREATE TABLE IF NOT EXISTS todos (
      id BIGINT PRIMARY KEY,
      text TEXT NOT NULL,
      done BOOLEAN DEFAULT false,
      added_at TEXT,
      done_at TEXT,
      notes TEXT DEFAULT '',
      url TEXT DEFAULT '',
      starred BOOLEAN DEFAULT false,
      tab TEXT DEFAULT 'personal',
      picks_order INT DEFAULT -1,
      last_date TEXT
    )
  `;
  // Add new columns if upgrading from old schema
  await sql`ALTER TABLE todos ADD COLUMN IF NOT EXISTS url TEXT DEFAULT ''`;
  await sql`ALTER TABLE todos ADD COLUMN IF NOT EXISTS starred BOOLEAN DEFAULT false`;
  await sql`ALTER TABLE todos ADD COLUMN IF NOT EXISTS tab TEXT DEFAULT 'personal'`;
}

exports.handler = async (event) => {
  const headers = {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Headers': 'Content-Type',
    'Content-Type': 'application/json',
  };

  if (event.httpMethod === 'OPTIONS') {
    return { statusCode: 200, headers, body: '' };
  }

  try {
    await ensureSchema();

    if (event.httpMethod === 'GET') {
      const tab = event.queryStringParameters?.tab || 'personal';
      const rows = await sql`
        SELECT * FROM todos WHERE tab = ${tab} ORDER BY id ASC
      `;

      if (rows.length === 0) {
        return { statusCode: 200, headers, body: JSON.stringify({ items: [], picks: [], lastDate: null }) };
      }

      const lastDate = rows[0].last_date || null;
      const picks = rows
        .filter(r => r.picks_order >= 0)
        .sort((a, b) => a.picks_order - b.picks_order)
        .map(r => Number(r.id));

      const items = rows.map(r => ({
        id: Number(r.id),
        text: r.text,
        done: r.done,
        addedAt: r.added_at,
        doneAt: r.done_at,
        notes: r.notes || '',
        url: r.url || '',
        starred: r.starred || false,
      }));

      return { statusCode: 200, headers, body: JSON.stringify({ items, picks, lastDate }) };
    }

    if (event.httpMethod === 'POST') {
      const body = JSON.parse(event.body || '{}');

      if (body.action === 'save_all') {
        const { items, picks, lastDate, tab = 'personal' } = body;

        // Delete removed items for this tab only
        const incomingIds = items.map(i => i.id);
        if (incomingIds.length > 0) {
          await sql`DELETE FROM todos WHERE tab = ${tab} AND id != ALL(${incomingIds})`;
        } else {
          await sql`DELETE FROM todos WHERE tab = ${tab}`;
        }

        // Upsert all items
        for (const item of items) {
          const picksOrder = picks.indexOf(item.id);
          await sql`
            INSERT INTO todos (id, text, done, added_at, done_at, notes, url, starred, tab, picks_order, last_date)
            VALUES (
              ${item.id}, ${item.text}, ${item.done}, ${item.addedAt},
              ${item.doneAt || null}, ${item.notes || ''}, ${item.url || ''},
              ${item.starred || false}, ${tab}, ${picksOrder}, ${lastDate}
            )
            ON CONFLICT (id) DO UPDATE SET
              text = EXCLUDED.text,
              done = EXCLUDED.done,
              added_at = EXCLUDED.added_at,
              done_at = EXCLUDED.done_at,
              notes = EXCLUDED.notes,
              url = EXCLUDED.url,
              starred = EXCLUDED.starred,
              tab = EXCLUDED.tab,
              picks_order = EXCLUDED.picks_order,
              last_date = EXCLUDED.last_date
          `;
        }

        return { statusCode: 200, headers, body: JSON.stringify({ ok: true }) };
      }
    }

    return { statusCode: 405, headers, body: JSON.stringify({ error: 'Method not allowed' }) };

  } catch (err) {
    console.error('DB error:', err);
    return { statusCode: 500, headers, body: JSON.stringify({ error: err.message }) };
  }
};
