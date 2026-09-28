# dhan-ai-skills

Kumpulan AI agent skills buatan [Dhanfinix](https://github.com/Dhanfinix) — reusable instruction sets untuk coding agent (Claude Code, Cursor, Codex, Gemini CLI, Antigravity, dll).

Install via [skills.sh](https://skills.sh):

```bash
npx skills add Dhanfinix/dhan-ai-skills
```

## Skills

| Skill | Deskripsi | Install |
|---|---|---|
| [android-ui-test](skills/android-ui-test/) | Author, fix, dan maintain UI test Android (XML Views & Jetpack Compose): Espresso/Compose in-app, UiAutomator cross-app, Maestro journey, cross-platform parity Android+iOS, CI wiring | `npx skills add Dhanfinix/dhan-ai-skills --skill android-ui-test` |

## Instalasi

### Semua skill

```bash
npx skills add Dhanfinix/dhan-ai-skills
```

### Skill tertentu

```bash
npx skills add Dhanfinix/dhan-ai-skills --skill android-ui-test
```

### Global (semua project, bukan cuma project saat ini)

```bash
npx skills add Dhanfinix/dhan-ai-skills --skill android-ui-test -g
```

### Non-interaktif (untuk onboarding script / CI)

```bash
npx skills add Dhanfinix/dhan-ai-skills --skill android-ui-test -g -y
```

### List skill yang tersedia tanpa install

```bash
npx skills add Dhanfinix/dhan-ai-skills --list
```

CLI otomatis mendeteksi agent yang terpasang dan menaruh skill di path yang benar (`.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, dst). Update setelah repo berubah: `npx skills update android-ui-test`.

## Struktur repo

```
skills/
└── android-ui-test/
    ├── SKILL.md              # entry point: decision tree layer testing + workflow agent
    └── refs/
        ├── espresso-views.md     # Espresso untuk screen XML Views
        ├── compose-testing.md    # Compose testing via semantics tree
        ├── hybrid-apps.md        # View + Compose, testTagsAsResourceId
        ├── uiautomator.md        # cross-app sederhana (dialog, notifikasi)
        ├── maestro.md            # journey cross-app: deeplink, share, OAuth
        ├── cross-platform.md     # parity Android + iOS dengan YAML bersama
        └── ci.md                 # GMD, GitHub Actions, Maestro Cloud, FTL
```

## Menambah skill baru

1. Buat folder baru di `skills/<nama-skill>/` berisi `SKILL.md` (YAML frontmatter `name` + `description` wajib).
2. Modul referensi opsional di `refs/`.
3. Push — dev lain tinggal `npx skills update`.

## Lisensi

MIT
