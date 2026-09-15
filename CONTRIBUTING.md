# Contributing to zinc-design

This repository holds **language design** and the **user manual** for [Zinc](https://github.com/zinc-lang/zinc).

Compiler, standard library, and tooling live in [zinc-lang/zinc](https://github.com/zinc-lang/zinc). Open compiler bugs and implementation PRs there. Open language-design changes here.

## Source of truth

- `user-manual/chinese/` is the **source of truth**.
- Translations must follow the Chinese text. If a translation disagrees with Chinese, fix the translation — or open an issue against the Chinese source if the source is wrong.
- Language design changes belong in `proposals/`, not only in the manual.

## What to work on

Good contributions, in order of how easily they land:

1. Review open translation PRs and keep editions aligned after Chinese edits.
2. Fill documented `TODO`s in the Chinese manual (then update every translation).
3. Write a proposal from `proposals/00-template.zh.md` or `proposals/00-template.en.md`.
4. Improve the root README, glossary, or chapter cross-links.

Do **not** mix a translation of one language with unrelated README or tooling changes. One language (or one proposal topic) per pull request.

## User manual layout

Chapter files share a numeric prefix so translations stay aligned:

| Prefix | Chinese (source) | Translation filename |
|--------|------------------|----------------------|
| 0 | `0_前言.md` | `0_preface.md` |
| 1 | `1_基础知识.md` | `1_basics.md` |
| 2 | `2_数据类型.md` | `2_data_types.md` |
| 3 | `3_函数、语句和表达式.md` | `3_functions_statements_expressions.md` |
| 4 | `4_动态内存分配和指针.md` | `4_memory_management.md` |
| 5 | `5_模式匹配.md` | `5_pattern_matching.md` |
| 6 | `6_trait和泛型.md` | `6_traits_and_generics.md` |
| 7 | `7_并行并发.md` | `7_parallelism_and_concurrency.md` |
| 8 | `8_unsafe和互操作.md` | `8_unsafe_and_interop.md` |
| 9 | `9_模块系统.md` | `9_module_system.md` |
| 99 | `99_编译器实现.md` | `99_compiler_implementation.md` |

Each translated directory should include a `README.md` that states Chinese is the source of truth and links every chapter.

## Translation rules

1. **Do not change Zinc syntax or identifiers.** Keep code fences, type names (`*T`, `*weak T`, `Send`, `Sync`), APIs (`box`, `std::ptr::upgrade`), and example typos unless the Chinese source is also being fixed.
2. **Translate diagram annotations**, but keep ASCII art and field names (`object_ptr`, `vtable_ptr`) intact.
3. **Keep `TODO` markers** and their meaning; translate the surrounding sentence.
4. Keep Zinc/Rust keywords in English: `unsafe`, `trait`, `box`, `borrow checker`, `Send`, `Sync`, `ABI`.
5. Glossary:

| Concept | Preferred English |
|---------|-------------------|
| 表传递 | dictionary passing |
| 实例化 | monomorphization |
| 引用计数 / ARC | reference counting / ARC (A = Automatic, not Atomic) |
| 借用 | borrow (types exist; there is **no borrow checker**) |
| 组件 | component (compile unit; analogous to a Rust crate) |
| `.zno` | `.zno` (SQLite “zinc oxide” interface file) |
| `*T: Send` | iff `T: Sync` (same for `&T` / `&mut T`) |

After translating, compare heading count, code-fence count, and `TODO` count with `user-manual/chinese/`.

## Proposals

New language-design writeups start from `proposals/00-template.zh.md` or `proposals/00-template.en.md`. Include Summary, Motivation, and Detailed Design. Discuss user-facing motivation first, then compiler/implementer details.

Some topics **require** a proposal before implementation, including unwind/exceptions, const generics, unsized types, and ABI-stable dynamic libraries. Chapter 99 lists questions that an unwind proposal must answer.

## Pull requests

- Fork the repo, branch from `main`, and open the PR against [zinc-lang/zinc-design](https://github.com/zinc-lang/zinc-design).
- Link the corresponding tracking issue.
- Mention review nits (terminology, TOC format) in the PR body instead of silently rewriting the source document.
