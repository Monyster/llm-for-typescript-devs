# Модуль 21: Voice та Realtime API — Голосові агенти

## 🎯 Що ви отримаєте з цього модуля

Після проходження ви будете:
- Розуміти архітектуру голосових AI-систем (STT → LLM → TTS)
- Використовувати OpenAI Realtime API для двостороннього голосового зв'язку
- Інтегрувати Speech-to-Text та Text-to-Speech у свої застосунки
- Будувати голосових агентів з function calling

**Які задачі це дозволяє вирішувати:** Голосові AI-асистенти для кол-центрів, voice-first інтерфейси для мобільних додатків, автоматичні телефонні системи, голосове керування IoT-пристроями.

---

## 21.1 Архітектура голосових AI-систем

### Класичний підхід: Pipeline

```
Голос → [STT] → Текст → [LLM] → Текст → [TTS] → Голос
         ↑                                  ↑
     Whisper/                          ElevenLabs/
     Deepgram                          OpenAI TTS
```

Три окремі кроки, кожен додає затримку (200-500ms на крок).

### Сучасний підхід: OpenAI Realtime API

```
Голос ←→ [Realtime API] ←→ Голос
          (все в одному)
          Латентність: ~300ms
```

Одна WebSocket-сесія, модель напряму обробляє та генерує аудіо.

---

## 21.2 Speech-to-Text: Голос → Текст

### OpenAI Whisper

```typescript
import OpenAI from 'openai';
import { createReadStream } from 'fs';

const client = new OpenAI();

// Транскрипція аудіо файлу
const transcription = await client.audio.transcriptions.create({
  model: 'whisper-1',
  file: createReadStream('./meeting-recording.mp3'),
  language: 'uk',  // Українська
  response_format: 'verbose_json',  // Включає timestamps
});

console.log(transcription.text);
// "Привіт, сьогодні ми обговоримо план на наступний квартал..."

// З timestamps
for (const segment of transcription.segments!) {
  console.log(`[${segment.start.toFixed(1)}s] ${segment.text}`);
}
```

### Deepgram (альтернатива, realtime STT)

```typescript
// Deepgram — streaming STT (реальний час)
import { createClient, LiveTranscriptionEvents } from '@deepgram/sdk';

const deepgram = createClient(process.env.DEEPGRAM_API_KEY!);
const connection = deepgram.listen.live({
  model: 'nova-2',
  language: 'uk',
  smart_format: true,
  interim_results: true,
});

connection.on(LiveTranscriptionEvents.Transcript, (data) => {
  const transcript = data.channel.alternatives[0].transcript;
  if (transcript) {
    console.log(`[Живий текст]: ${transcript}`);
  }
});

// Надсилаємо аудіо-потік
microphone.on('data', (audioChunk: Buffer) => {
  connection.send(audioChunk);
});
```

---

## 21.3 Text-to-Speech: Текст → Голос

### OpenAI TTS

```typescript
import OpenAI from 'openai';
import { writeFileSync } from 'fs';

const client = new OpenAI();

const response = await client.audio.speech.create({
  model: 'tts-1-hd',     // 'tts-1' — швидший, 'tts-1-hd' — якісніший
  voice: 'nova',          // alloy, echo, fable, onyx, nova, shimmer
  input: 'Привіт! Я ваш AI-асистент. Чим можу допомогти?',
  response_format: 'mp3',
  speed: 1.0,             // 0.25 - 4.0
});

// Збереження у файл
const buffer = Buffer.from(await response.arrayBuffer());
writeFileSync('response.mp3', buffer);
```

### ElevenLabs (найякісніший голос)

```typescript
// ElevenLabs — найреалістичніші голоси, підтримка клонування
const response = await fetch(
  `https://api.elevenlabs.io/v1/text-to-speech/${voiceId}`,
  {
    method: 'POST',
    headers: {
      'xi-api-key': process.env.ELEVENLABS_API_KEY!,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      text: 'Привіт! Як справи?',
      model_id: 'eleven_multilingual_v2',
      voice_settings: {
        stability: 0.5,
        similarity_boost: 0.75,
      },
    }),
  }
);

const audioBuffer = await response.arrayBuffer();
```

---

## 21.4 OpenAI Realtime API — все в одному

Realtime API — це WebSocket-з'єднання де модель одночасно слухає та говорить:

```typescript
import OpenAI from 'openai';

const client = new OpenAI();

// Створення Realtime сесії
const session = await client.realtime.sessions.create({
  model: 'gpt-4o-realtime-preview',
  voice: 'nova',
  instructions: `Ти — AI-асистент підтримки для TechShop.
Говори українською. Будь дружнім та лаконічним.
Можеш перевіряти замовлення та відповідати на питання.`,
  tools: [
    {
      type: 'function',
      name: 'check_order',
      description: 'Перевірити статус замовлення',
      parameters: {
        type: 'object',
        properties: {
          order_id: { type: 'string' },
        },
        required: ['order_id'],
      },
    },
  ],
  turn_detection: {
    type: 'server_vad',  // Автоматичне визначення коли людина закінчила говорити
    threshold: 0.5,
    prefix_padding_ms: 300,
    silence_duration_ms: 500,
  },
});

// WebSocket з'єднання
const ws = new WebSocket(
  'wss://api.openai.com/v1/realtime',
  { headers: { 'Authorization': `Bearer ${process.env.OPENAI_API_KEY}` } }
);

ws.on('message', (data) => {
  const event = JSON.parse(data.toString());

  switch (event.type) {
    case 'response.audio.delta':
      // Аудіо-відповідь від моделі — відтворити на динаміку
      playAudio(Buffer.from(event.delta, 'base64'));
      break;

    case 'response.function_call_arguments.done':
      // Модель хоче викликати функцію
      const result = await handleToolCall(event.name, JSON.parse(event.arguments));
      // Відправити результат назад
      ws.send(JSON.stringify({
        type: 'conversation.item.create',
        item: {
          type: 'function_call_output',
          call_id: event.call_id,
          output: JSON.stringify(result),
        },
      }));
      break;

    case 'input_audio_buffer.speech_started':
      console.log('Користувач почав говорити...');
      break;

    case 'input_audio_buffer.speech_stopped':
      console.log('Користувач закінчив говорити');
      break;
  }
});

// Надсилаємо аудіо з мікрофону
microphone.on('data', (chunk: Buffer) => {
  ws.send(JSON.stringify({
    type: 'input_audio_buffer.append',
    audio: chunk.toString('base64'),
  }));
});
```

### Вартість Realtime API

| Компонент | Ціна |
|-----------|------|
| Аудіо вхід | $0.06 / хвилина |
| Аудіо вихід | $0.24 / хвилина |
| Текстовий вхід | $5.00 / 1M токенів |
| Текстовий вихід | $20.00 / 1M токенів |

Типовий 5-хвилинний дзвінок: ~$1.50 (дешевше за оператора кол-центру).

---

## 21.5 Pipeline підхід: STT + LLM + TTS

Для більшого контролю та нижчої вартості:

```typescript
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
import OpenAI from 'openai';

const client = new OpenAI();

async function voiceAssistant(audioInput: Buffer): Promise<Buffer> {
  // 1. STT: голос → текст
  const transcription = await client.audio.transcriptions.create({
    model: 'whisper-1',
    file: new File([audioInput], 'input.wav', { type: 'audio/wav' }),
    language: 'uk',
  });
  console.log(`[STT] ${transcription.text}`);

  // 2. LLM: текст → відповідь
  const { text } = await generateText({
    model: openai('gpt-4o-mini'),
    system: 'Ти — голосовий асистент. Відповідай коротко (1-2 речення).',
    prompt: transcription.text,
  });
  console.log(`[LLM] ${text}`);

  // 3. TTS: текст → голос
  const speech = await client.audio.speech.create({
    model: 'tts-1',
    voice: 'nova',
    input: text,
    response_format: 'mp3',
  });
  console.log('[TTS] Аудіо згенеровано');

  return Buffer.from(await speech.arrayBuffer());
}
```

---

## 21.6 Порівняння підходів

| Критерій | Pipeline (STT+LLM+TTS) | OpenAI Realtime |
|----------|------------------------|-----------------|
| Латентність | 1-3 секунди | ~300ms |
| Вартість | Нижча (~$0.50/5хв) | Вища (~$1.50/5хв) |
| Контроль | Повний (кожен крок окремо) | Обмежений |
| Переривання | Складно реалізувати | Вбудоване |
| Провайдер-агностичність | Так (різні STT/TTS) | Тільки OpenAI |
| Function calling | Через LLM | Вбудоване |

---

## Перевір себе

1. Чим відрізняється Pipeline підхід від Realtime API?
2. Коли Pipeline краще за Realtime? Коли навпаки?
3. Скільки коштує 10-хвилинний голосовий дзвінок через Realtime API?
4. Реалізуйте STT → LLM → TTS pipeline для простого асистента
5. Як додати function calling до голосового агента?

---

**Назад:** [← Модуль 20 — Fine-tuning](20-fine-tuning.md) | **Далі:** [Модуль 22 — AI SDK UI глибше →](22-ai-sdk-ui.md)
