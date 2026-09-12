# AGENTS.md

This repository holds the **design documents** and **user manual** for [Zinc](https://github.com/zinc-lang): a Rust-inspired systems language that keeps memory/thread safety, drops the borrow checker, and implements generics primarily via dictionary passing.

## Source of truth

- The Chinese user manual in `user-manual/chinese/` is the **source of truth**.
- Translations must follow the Chinese text. If a translation disagrees with Chinese, fix the translation (or open an issue against the Chinese source if the source itself is wrong).
- Language design changes belong in `proposals/`, not only in the manual.

## Layout

```
user-manual/chinese/     # source manual (chapter files in Chinese)
user-manual/english/     # English translation
user-manual/spanish/     # Spanish translation
user-manual/french/      # French translation
user-manual/japanese/    # Japanese translation
user-manual/korean/      # Korean translation
proposals/               # design proposals (`00-template.zh.md` / `00-template.en.md`)
assets/                  # logo and shared images
```

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

1. **Do not change Zinc syntax or identifiers.** Keep code fences, type names (`*T`, `*weak T`, `Send`, `Sync`), APIs (`std::ptr::upgrade`, `box`), and source typos that appear in examples (for example `MyBussinessError`) unless the Chinese source is also being fixed.
2. **Translate diagram annotations**, but keep ASCII art and field names (`object_ptr`, `vtable_ptr`) intact.
3. **Keep `TODO` markers** and their meaning; translate the surrounding sentence.
4. **Prefer established systems-programming terms** in the target language, and keep Zinc/Rust keywords in English: `unsafe`, `trait`, `box`, `borrow checker`, `Send`, `Sync`, `ABI`.
5. Use this glossary for core concepts:

| Concept | Preferred English | Notes |
|---------|-------------------|--------|
| 表传递 | dictionary passing | Also called table passing; English should use **dictionary passing** |
| 实例化 | monomorphization | Compiler option vs default dictionary passing |
| 引用计数 / ARC | reference counting / ARC | A = Automatic (atomic vs non-atomic), not Atomic |
| 借用 | borrow | Types exist; there is **no borrow checker** |
| 组件 | component | Compile unit; analogous to a Rust crate |
| `.zno` | `.zno` | “Zinc oxide” header/interface file (SQLite) |

6. After translating, compare heading count, code-fence count, and `TODO` count with `user-manual/chinese/`.

## Proposals

New language-design writeups start from `proposals/00-template.zh.md` or `proposals/00-template.en.md`. Include Summary, Motivation, and Detailed Design. Discuss user-facing motivation first, then compiler/implementer details.

## Pull requests

- One language (or one proposal topic) per PR.
- Do not mix translation work with unrelated README/agent/skill changes.
- Link the corresponding tracking issue.
- Mention review nits (terminology consistency, untranslated comments that exist only in the source, TOC format) in the PR body instead of silently rewriting the source document.

When working on translations, also read `.cursor/skills/zinc-docs-translation/SKILL.md`.
