---
excerpt: ""
title: "Migrating from fp-ts to Effect"
tags: [fp-ts, effect, typescript]
author: addamsson
short_title: "Migrating from fp-ts to Effect"
comments: true
updated_at: 2023-08-05
series: effect
draft: true
---

> 

## Refactor your fp-ts code
- migrate everything to `ReaderTaskEither`
- Use `W` versions of everything, eg: `chainW` (or better yet `flatMap`), `bindW`, etc.
- import the specific functions and alias them, eg: `import { bindW as bind } from 'fp-ts/lib/ReaderTaskEither'`

## RTE mappings
- `right` -> `succeed`` or `sync`
- `chain` -> `flatMap`

## Do notation
- Use `Effect.Do.pipe` that combines `pipe` and `RTE.Do`

## Conclusion



Until then, let's go forth and **kode on**!