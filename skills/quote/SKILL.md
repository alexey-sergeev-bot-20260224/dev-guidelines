---
name: quote
description: "Fetch random inspirational or themed quotes. Use when the user asks for a quote, motivation, inspiration, or a 'quote of the day'. No API key needed."
metadata: { "openclaw": { "emoji": "💬", "requires": { "bins": ["curl"] } } }
---

# Quote Skill

Fetch random quotes from free APIs.

## When to Use

✅ **USE this skill when:**

- "Give me a quote"
- "Inspirational quote"
- "Quote of the day"
- "Motivate me"
- "Random wisdom"

## Commands

### Random Quote (ZenQuotes API)

```bash
# Random quote (JSON array with one entry)
curl -s "https://zenquotes.io/api/random"
```

Response format: `[{"q":"Quote text","a":"Author","h":"HTML formatted"}]`

### Quote of the Day

```bash
curl -s "https://zenquotes.io/api/today"
```

### Multiple Random Quotes

```bash
# Returns 50 random quotes
curl -s "https://zenquotes.io/api/quotes"
```

## Formatting

When presenting a quote to the user, format it nicely:

> "Quote text here"
> — Author Name

Keep it clean and elegant. One quote at a time unless the user asks for more.

## Notes

- No API key required
- ZenQuotes has rate limiting — don't spam
- Quotes are in English
- If the API is down, apologize and offer to try again later
