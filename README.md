# Safety Guard — Pre-action decision for shell actions

> **Standalone.** Split from the [KutAI](https://github.com/msakirc/kutai) monorepo (`packages/safety_guard/`), where it is developed. It does not need the KutAI app; its test suite passes with only its declared dependencies installed.

[English](#english) · [Türkçe](#türkçe)

<a id="english"></a>

## Purpose

Before a mechanical workflow step runs a shell command, Mr. Roboto asks Safety Guard
for one decision: **Allow**, **WaitForFounder** or **Block**. Safety Guard combines a
hard blocklist (force-push to main/master, `vercel deploy --prod`, `aws s3 rm`,
`stripe.charges.create`, `rm -rf /`), collision guards (force push, rebase /
`reset --hard` on a shared branch, absolute paths outside the mission workspace,
`DROP TABLE` / `TRUNCATE` on a non-mission table) and the step's reversibility tag.

**It does NOT** sandbox or execute anything, and it is not a security boundary: the
checks are regular expressions over the command string, so an obfuscated command
can pass. It is a guard against an agent's honest mistakes, not against an attacker.
It performs no I/O; the caller supplies the branch, workspace and allowlist.

## Install

```bash
pip install "safety_guard @ git+https://github.com/msakirc/safety-guard.git"
```

Requires Python 3.10+. Sibling KutAI packages are installed from their own repositories.

## Quick start

```python
from safety_guard import pre_action

step = {"reversibility": "partial"}
for cmd in ("git push origin feature/login", "git push --force origin main"):
    decision = pre_action(step, {"command": cmd}, workspace_root=".",
                          current_branch="main", founder_recently_active=True)
    print(f"{cmd!r:40} -> {decision}")
```

## Public API

```python
from safety_guard import pre_action, Allow, WaitForFounder, Block, Reversibility, resolve

decision = pre_action(
    step,                                   # workflow step dict (reversibility, locked)
    {"command": "git push origin main"},
    workspace_root="/work/mission_12",
    current_branch="main",
    founder_recently_active=True,
    mission_allowlist=[r"^git push origin feature/"],   # regexes that bypass collision guards
    runtime_reversibility=None,             # executor may escalate, never downgrade
)                                           # -> Allow() | WaitForFounder(reason) | Block(reason)

tag = resolve(step, Reversibility.PARTIAL)  # -> effective Reversibility
```

Order: blocklist (always wins) → collision guards (bypassable only by the per-mission
allowlist) → reversibility. An `irreversible` step waits for the founder when the step
is `locked` or the founder is not recently active.

## Key Modules

| module | role |
|---|---|
| `executor_hook.py` | `pre_action` and the three decision types |
| `collision.py` | pure detectors + hard blocklist patterns |
| `tags.py` | `Reversibility` (full < partial < irreversible; `"none"` = legacy alias) and `resolve` |

## Gotchas

- `resolve`: the static tag in the workflow JSON is the floor; a runtime override may
  only make it stricter. `locked: true` ignores overrides entirely. Unknown labels
  default to `full` with a warning.
- `detect_destructive_shared_db` only matches a statement at the **start** of the
  string, and tables named `mission_<n>…` are allowed.
- The only caller today (`mr_roboto._safety_guard_check`) passes
  `founder_recently_active=True` until a real activity signal is wired, so
  irreversible steps only wait when they are `locked`.

## Tests

```powershell
python -m pytest packages/safety_guard/tests
```

## In KutAI

Mr. Roboto asks `pre_action()` before every mechanical shell step; `Block` fails the step and `WaitForFounder` parks it as `waiting_human`. Source: [`packages/safety_guard/`](https://github.com/msakirc/kutai/tree/main/packages/safety_guard) in the [KutAI monorepo](https://github.com/msakirc/kutai), where this package is developed.

## Status and limitations

- Standalone: it does not import the KutAI app, and its tests pass with only its declared dependencies.
- Version 0.1.0, part of a personal research project. Developed and tested on Windows 11 with Python 3.10; other platforms are untested.
- A pattern blocklist, not a sandbox: it inspects the command string and does not isolate execution.

## License

MIT. See [LICENSE](https://github.com/msakirc/kutai/blob/main/LICENSE).

---
<a id="türkçe"></a>
## Türkçe

### Amaç

Mekanik bir iş akışı adımı kabuk komutu çalıştırmadan önce Mr. Roboto, Safety
Guard'dan tek bir karar ister: **Allow**, **WaitForFounder** ya da **Block**. Safety
Guard kesin bir yasak listesini (main/master'a zorla gönderme, `vercel deploy --prod`,
`aws s3 rm`, `stripe.charges.create`, `rm -rf /`), çakışma korumalarını (zorla
gönderme, paylaşılan dalda rebase / `reset --hard`, görev çalışma alanı dışındaki
mutlak yollar, görev dışı tabloda `DROP TABLE` / `TRUNCATE`) ve adımın geri
alınabilirlik etiketini birleştirir.

Hiçbir şeyi **yalıtmaz ya da çalıştırmaz** ve bir güvenlik sınırı değildir: denetimler
komut metni üzerinde düzenli ifadelerdir, gizlenmiş bir komut geçebilir. Bir saldırgana
karşı değil, ajanın dürüst hatalarına karşı bir korumadır. G/Ç yapmaz; dalı, çalışma
alanını ve izin listesini çağıran verir.

### Kurulum

```bash
pip install "safety_guard @ git+https://github.com/msakirc/safety-guard.git"
```

Python 3.10+ gerekir. Kardeş KutAI paketleri kendi depolarından kurulur.

### Hızlı başlangıç

```python
from safety_guard import pre_action

step = {"reversibility": "partial"}
for cmd in ("git push origin feature/login", "git push --force origin main"):
    decision = pre_action(step, {"command": cmd}, workspace_root=".",
                          current_branch="main", founder_recently_active=True)
    print(f"{cmd!r:40} -> {decision}")
```

### Genel API

```python
from safety_guard import pre_action, Allow, WaitForFounder, Block

decision = pre_action(step, {"command": "git push origin main"},
                      workspace_root="/work/mission_12", current_branch="main")
# -> Allow() | WaitForFounder(reason) | Block(reason)
```

### Dikkat Edilecekler

- Statik etiket tabandır; çalışma zamanı geçersiz kılması yalnızca daha sıkı yapabilir.
  `locked: true` geçersiz kılmaları tamamen yok sayar.
- Bugünkü tek çağıran `founder_recently_active=True` geçirir; geri alınamaz adımlar
  yalnızca `locked` olduklarında bekler.

### Testler

```powershell
python -m pytest packages/safety_guard/tests
```

### KutAI içindeki yeri

Mr. Roboto her mekanik shell adımından önce `pre_action()`'a sorar; `Block` adımı başarısız kılar, `WaitForFounder` ise onu `waiting_human` durumunda bekletir. Kaynak: [KutAI monorepo](https://github.com/msakirc/kutai) içinde [`packages/safety_guard/`](https://github.com/msakirc/kutai/tree/main/packages/safety_guard); paket orada geliştirilir.

### Durum ve sınırlamalar

- Bağımsız: KutAI uygulamasını import etmez; testleri yalnızca bildirilen bağımlılıklarla geçer.
- Sürüm 0.1.0, kişisel bir araştırma projesinin parçası. Windows 11 ve Python 3.10 üzerinde geliştirilip test edildi; diğer platformlar denenmedi.
- Bir kalıp engel listesidir, sandbox değildir: komut metnine bakar, yürütmeyi yalıtmaz.

### Lisans

MIT. [LICENSE](https://github.com/msakirc/kutai/blob/main/LICENSE) dosyasına bakın.
