# URL Shortener Design Review (TinyURL)

## Summary

This is my simplified review of a URL shortener design (like TinyURL). The goal is to understand the core system, what works well, and what still needs work for real production use.

## Problem

We need a service that:

1. Takes a long URL and gives back a short URL.
2. Redirects users from the short URL to the original long URL.

Example:

- Long: https://www.example.com/very/long/url
- Short: https://tinyurl.com/abc123

## Scale Assumptions

| Requirement | Estimate |
|-------------|----------|
| New URLs per day | 100 million |
| New URLs per second | about 1,160 |
| Reads per second (10:1 read/write) | about 11,600 |
| Total URLs in 10 years | 365 billion |
| Storage in 10 years | about 365 TB |

Other requirements:

- Short links should be compact.
- Allowed characters: a-z, A-Z, 0-9.
- High availability.
- No update/delete flow in this version.

## Basic API

| Action | Method | Endpoint | Purpose |
|--------|--------|----------|---------|
| Create short URL | POST | /api/v1/data/shorten | Return a short URL for a long URL |
| Redirect | GET | /api/v1/shortUrl | Redirect to long URL |

## Redirect Choice: 301 vs 302

- 301 (permanent): better for reducing server load because browsers cache redirect behavior.
- 302 (temporary): better if we want accurate click tracking through our service.

For analytics, 302 is usually preferred.

## Storage Model

Use both:

- Database for durable short -> long mapping.
- Cache for fast read performance.

Simple table:

| Column | Description |
|--------|-------------|
| id | Unique numeric ID |
| shortURL | Short code |
| longURL | Original URL |

## Short URL Generation Options

### Option 1: Hash and Resolve Collisions

1. Hash long URL.
2. Use first N characters.
3. If already taken, retry with a variation.

Pros:

- No central ID service needed.

Cons:

- Collisions and retries add overhead.

### Option 2: Base62 from Unique ID

1. Generate a unique numeric ID.
2. Convert ID to Base62 (0-9, a-z, A-Z).

Example: 2009215674938 -> zn9edcu

Pros:

- Predictable and collision-free if IDs are unique.

Cons:

- Requires reliable distributed ID generation.

Chosen approach: Base62 from unique ID.

## Request Flows

### Shorten flow

1. Client sends long URL.
2. Check if URL already exists.
3. If it exists, return existing short URL.
4. If not, create ID, convert to Base62, store mapping.
5. Return short URL.

### Redirect flow

1. Client requests short URL.
2. Check cache first.
3. If miss, read from database.
4. Store result in cache.
5. Return 302 redirect to long URL.

## What Is Good

- Clear logic and realistic scale estimates.
- Good comparison of alternatives.
- Practical use of cache for read-heavy traffic.

## Gaps for Real Production

- No rate limiting (abuse risk).
- No detailed sharding plan for very large data.
- Cache eviction/failure behavior not defined.
- Sequential IDs can make URL guessing easier.
- Analytics pipeline is not fully designed.
- Distributed ID generation is not explained enough.

## Final Take

This is a strong interview-level design and a good foundation. To make it production-ready, we should add abuse protection, stronger scaling strategy, security hardening, and a detailed analytics pipeline.