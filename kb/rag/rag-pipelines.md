# Модуль 35: Data pipelines для RAG — Індексація, оновлення, версіонування

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Будувати automated pipeline: нові документи → chunking → embedding → index
- Реалізовувати incremental indexing (тільки змінене)
- Обробляти різні формати: PDF, DOCX, HTML, Markdown, CSV
- Версіонувати індекс та робити rollback

**Які задачі це дозволяє вирішувати:** RAG з Модуля 10 працює зі статичними даними. У реальності дані оновлюються: нові статті, змінені документи, видалені файли. Потрібен pipeline що тримає індекс актуальним.

---

## 35.1 Архітектура RAG Pipeline

```
Джерела даних                  Pipeline                   Індекс
┌──────────────┐           ┌────────────────┐        ┌──────────┐
│ Google Drive │──┐        │ 1. Detect      │        │ pgvector │
│ Confluence   │──┤───────▶│ 2. Extract     │───────▶│ / Pinecone│
│ GitHub Wiki  │──┤        │ 3. Chunk       │        └──────────┘
│ Notion       │──┤        │ 4. Embed       │
│ Local files  │──┘        │ 5. Store       │
└──────────────┘           └────────────────┘
                                 ↓
                           ┌────────────────┐
                           │ 6. Cleanup     │
                           │ (видалені docs)│
                           └────────────────┘
```

---

## 35.2 Document Loader: Різні формати

```typescript
import { readFile } from 'fs/promises';

interface Document {
  id: string;
  content: string;
  metadata: {
    source: string;
    format: string;
    lastModified: Date;
    checksum: string;
  };
}

async function loadDocument(filepath: string): Promise<Document> {
  const ext = filepath.split('.').pop()?.toLowerCase();
  let content: string;

  switch (ext) {
    case 'md':
    case 'txt':
      content = await readFile(filepath, 'utf-8');
      break;

    case 'pdf':
      const pdfParse = (await import('pdf-parse')).default;
      const pdfBuffer = await readFile(filepath);
      const pdf = await pdfParse(pdfBuffer);
      content = pdf.text;
      break;

    case 'docx':
      const mammoth = await import('mammoth');
      const docxBuffer = await readFile(filepath);
      const result = await mammoth.extractRawText({ buffer: docxBuffer });
      content = result.value;
      break;

    case 'html':
      const htmlContent = await readFile(filepath, 'utf-8');
      content = htmlContent.replace(/<[^>]*>/g, ' ').replace(/\s+/g, ' ').trim();
      break;

    case 'csv':
      content = await readFile(filepath, 'utf-8');
      // CSV → рядки як окремі записи
      break;

    default:
      throw new Error(`Unsupported format: ${ext}`);
  }

  const checksum = createHash('sha256').update(content).digest('hex');

  return {
    id: filepath,
    content,
    metadata: {
      source: filepath,
      format: ext!,
      lastModified: (await stat(filepath)).mtime,
      checksum,
    },
  };
}
```

---

## 35.3 Incremental Indexing

Переіндексовуємо тільки змінені документи:

```typescript
import { createHash } from 'crypto';

class IncrementalIndexer {
  // Зберігаємо checksums у БД
  async getStoredChecksum(docId: string): Promise<string | null> {
    const row = await db.query('SELECT checksum FROM document_index WHERE doc_id = $1', [docId]);
    return row.rows[0]?.checksum ?? null;
  }

  async processDocuments(documents: Document[]) {
    let added = 0, updated = 0, skipped = 0;

    for (const doc of documents) {
      const storedChecksum = await this.getStoredChecksum(doc.id);

      if (storedChecksum === doc.metadata.checksum) {
        skipped++; // Документ не змінився — пропускаємо
        continue;
      }

      if (storedChecksum) {
        // Документ змінився — видаляємо старі chunks
        await this.deleteChunks(doc.id);
        updated++;
      } else {
        added++;
      }

      // Chunking + Embedding + Store
      const chunks = chunkDocument(doc.content, doc.metadata);
      const embeddings = await embedMany({
        model: openai.embedding('text-embedding-3-small'),
        values: chunks.map(c => c.text),
      });

      for (let i = 0; i < chunks.length; i++) {
        await db.query(`
          INSERT INTO chunks (id, doc_id, content, metadata, embedding)
          VALUES ($1, $2, $3, $4, $5)
        `, [`${doc.id}:${i}`, doc.id, chunks[i].text, chunks[i].metadata, embeddings.embeddings[i]]);
      }

      // Оновлюємо checksum
      await db.query(`
        INSERT INTO document_index (doc_id, checksum, chunk_count, indexed_at)
        VALUES ($1, $2, $3, NOW())
        ON CONFLICT (doc_id) DO UPDATE SET checksum = $2, chunk_count = $3, indexed_at = NOW()
      `, [doc.id, doc.metadata.checksum, chunks.length]);
    }

    return { added, updated, skipped };
  }

  // Видалення документів яких більше не існує
  async cleanupDeleted(currentDocIds: Set<string>) {
    const indexed = await db.query('SELECT doc_id FROM document_index');
    let deleted = 0;

    for (const row of indexed.rows) {
      if (!currentDocIds.has(row.doc_id)) {
        await this.deleteChunks(row.doc_id);
        await db.query('DELETE FROM document_index WHERE doc_id = $1', [row.doc_id]);
        deleted++;
      }
    }

    return { deleted };
  }

  private async deleteChunks(docId: string) {
    await db.query('DELETE FROM chunks WHERE doc_id = $1', [docId]);
  }
}
```

---

## 35.4 Автоматичний запуск

```typescript
import cron from 'node-cron';

const indexer = new IncrementalIndexer();

// Кожну годину перевіряємо зміни
cron.schedule('0 * * * *', async () => {
  console.log('[RAG Pipeline] Starting incremental index...');

  const documents = await scanDocumentSources([
    { type: 'local', path: './docs' },
    { type: 'github', repo: 'company/wiki' },
  ]);

  const result = await indexer.processDocuments(documents);
  const cleanup = await indexer.cleanupDeleted(new Set(documents.map(d => d.id)));

  console.log(`[RAG Pipeline] Done: +${result.added} ~${result.updated} =${result.skipped} -${cleanup.deleted}`);
});
```

---

## 35.5 Версіонування індексу

```typescript
// Snapshot перед великими змінами
async function createIndexSnapshot(version: string) {
  await db.query(`
    CREATE TABLE chunks_snapshot_${version} AS SELECT * FROM chunks
  `);
  await db.query(`
    INSERT INTO index_versions (version, created_at, chunk_count)
    SELECT $1, NOW(), COUNT(*) FROM chunks
  `, [version]);
}

// Rollback якщо щось пішло не так
async function rollbackIndex(version: string) {
  await db.query('TRUNCATE chunks');
  await db.query(`INSERT INTO chunks SELECT * FROM chunks_snapshot_${version}`);
}
```

---

## Перевір себе

1. Чому incremental indexing краще за повну переіндексацію?
2. Як визначити що документ змінився (checksum)?
3. Реалізуйте pipeline для папки з Markdown файлами
4. Як обробити видалення документа (cleanup)?
5. Навіщо версіонування індексу і коли робити snapshot?

---

**Назад:** [← Модуль 34 — Rate limiting](34-rate-limiting.md) | **Далі:** [Модуль 36 — Compliance та аудит →](36-compliance.md)
