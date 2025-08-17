## 3. Как состояния процессов влияют на подсистемы энергосбережения (PM)

Ниже — практический взгляд на связь «состояния задач ↔ решения PM». Под PM понимаем: **cpuidle** (C-состояния), **cpufreq** (частота/напряжение, P-состояния), системные сны (**s2idle/standby/S3/S4**), **runtime PM** устройств, а также подсказки планировщику (**PELT/uclamp/EAS**).

### Картина в целом

* Пока **есть runnable-нагрузка** (задачи в `TASK_RUNNING`/`on_cpu`), CPU не уходит в простой; его **utilization** по PELT остаётся выше нуля, что удерживает частоту выше и сдвигает cpuidle к **более мелким C-состояниям** (или вовсе не даёт уснуть). С `schedutil` частота напрямую следует за оценкой `util_avg` CPU/задач; при пробуждении после I/O действует ещё и **iowait-boost** (кратковременный подскок частоты для “встряхивания” I/O-навороченных задач). \[1]\[2]\[3]
* Когда **все задачи спят** (`TASK_INTERRUPTIBLE`/`TASK_UNINTERRUPTIBLE`/ожидание событий), ядро входит в **idle-петлю**, может **остановить тик** (NOHZ idle) и выбрать **глубокие C-состояния** по governor’у `menu`/`teo`, ориентируясь на прогноз длительности простоя и ближайшее событие таймера. Это резко снижает потребление. \[4]\[5]
* Для системного сна (s2idle/standby/S3/S4) пользовательские процессы и часть kthread’ов переводятся freezer’ом в **`TASK_FROZEN`**, что гарантирует отсутствие активности на время перехода; параллельно драйверы уводят устройства в минимальные состояния. **Активные wakeup-источники** (wakelocks) или незавершённые I/O могут **запретить** вход в сон. \[6]\[7]\[8]

### Что делает **каждое состояние** с PM

* **`TASK_RUNNING` / `on_cpu`**
  Увеличивает `util_avg` у сущности и у CPU (PELT), что через `schedutil` поднимает частоту и удерживает её выше (частично сглаженно «памятью» PELT). Не даёт войти в idle/C-state. На big.LITTLE включается **EAS**: выбор CPU учитывает и производительность, и энергозатраты для данной `util_avg`. \[2]\[9]\[10]

* **`TASK_INTERRUPTIBLE`** (сон, прерываемый сигналом)
  Пока задача **не runnable**, её вклад в `util_avg` **экспоненциально затухает** (PELT-blocked), освобождая CPU для idle; таймеры и внешние IRQ — единственные источники пробуждения CPU. При пробуждении после I/O `schedutil` применит **iowait-boost**, чтобы не стартовать с заниженной частоты. \[2]\[3]\[5]

* **`TASK_UNINTERRUPTIBLE`** (обычно D-state, ожидание I/O/битов завершения)
  Аналогично, задача не runnable, позволяет уходить в idle. Однако с точки зрения **устройств**: драйвер мог взять **runtime-PM usage-count** или удерживать железо в активном состоянии, пока операция не завершится — тем самым мешая autosuspend устройства и (через wakeup-источники) системному сну. \[7]\[11]

* **`TASK_STOPPED`/`TASK_TRACED`**
  Не даёт runnable-нагрузки → способствует idle. Отладчики/ptrace могут, наоборот, **удерживать устройства** (tty/usb-serial и пр.) активными через открытые дескрипторы — это уже про runtime PM. \[11]

* **`TASK_ZOMBIE`/`DEAD`**
  На PM не влияет; ресурсы освобождены, нагрузка нулевая.

* **`TASK_FROZEN`** (freezer при suspend/hibernate)
  Полностью исключает произвольные пробуждения задач в ходе перехода в сон; драйверы последовательно переводят устройства в низкие состояния; активные **wakeup-sources** блокируют переход (autosleep ждёт, пока «счётчик пробуждений» обнулится). \[6]\[8]

### Как **переходы между состояниями** перекладываются в решения PM

1. **Переход в сон (`RUNNING → *SLEEP`)**

   * **cpuidle**: тик может быть остановлен (NOHZ), governor выбирает C-state по прогнозу времени простоя (ближайший таймер, история). Чем меньше будущих событий (таймеры реже, IRQ тише), тем глубже C-state. \[4]\[5]
   * **cpufreq**: `util_avg` CPU/задач падает ⇒ `schedutil` снижает целевую частоту; при полном простое частота несущественна (ядро уже в C-state). \[2]

2. **Пробуждение по событию (`*SLEEP → RUNNABLE`)**

   * **cpufreq**: короткий **iowait-boost** ускоряет «разгон» частоты, чтобы I/O-bound-задачи не буксовали на низком P-state сразу после wakeup. \[3]
   * **EAS** (на гетерогенных CPU): планировщик может **переместить** задачу на «быстрый» или «экономичный» кластер в зависимости от `util_avg`/uclamp и модели энергии. \[9]\[10]\[12]

3. **Системный сон**

   * Вход: freezer переводит задачи в `TASK_FROZEN` → **устройства** последовательно уводятся в низкие состояния → платформа усыпляется (s2idle/standby/S3, либо S4). **Wakelocks**/активные wakeup-источники или неготовность драйверов **прерывают** переход. \[6]\[8]
   * Выход: thaw задач → возобновление таймеров/IRQ; PELT и `schedutil` постепенно «догоняют» актуальную активность. \[6]

### Подсказки и политики, влияющие через состояния

* **PELT + util\_est**: учитывают «блокированную» активность (blocked contribution), чтобы частота не падала слишком резко во время частых коротких снов. Это косвенно сглаживает влияние `SLEEP`↔`RUN` на P-state. \[2]
* **`uclamp` (UCLAMP\_MIN/MAX)**: задаёт **нижнюю/верхнюю** планку эффективности для задач/сгруппированных cgroup’ов. При `schedutil` это напрямую **клампит частоту**, даже если `util_avg` низок (например, для интерактивности). \[13]\[14]
* **cpusets/cgroup v2**: ограничивают, где задачи **могут** работать (ядра/NUMA), что влияет на формирование `util_avg` по CPU и на доступные C-/P-состояния кластера. \[15]\[16]

### Runtime PM и «состояния задач»

Даже когда задача «спит», **открытые дескрипторы/ongoing I/O** нередко держат устройство «вверх» (через `pm_runtime_get()`/usage-count), мешая его autosuspend. Обратное тоже верно: когда задача завершает I/O и драйвер делает `pm_runtime_put()`, устройство засыпает отдельно от CPU. Эти связи особенно заметны для NIC/USB/MMC/звук. \[11]

---

### Ссылки

\[1] Документация планировщика: capacity/PELT/schedutil. ([docs.kernel.org][1], [static.lwn.net][2])
\[2] PELT и сигналы загрузки (util\_avg) в CFS. ([docs.kernel.org][1])
\[3] I/O-wait boost в `schedutil` (обзор). ([review.lineageos.org][3])
\[4] NOHZ/tickless idle и остановка тика при простое. ([linux.kernel.narkive.com][4])
\[5] cpuidle и выбор C-состояния (`menu`/`teo`).
\[6] Freezer: `try_to_freeze()`, `TASK_FROZEN`, порядок suspend/resume. ([docs.kernel.org][5], [infradead.org][6])
\[7] Runtime PM устройств (usage-count, autosuspend). ([lkml.org][7])
\[8] Системные сны и интерфейсы `/sys/power/state`, `/sys/power/autosleep`, wakeup-sources. ([docs.kernel.org][8], [android.googlesource.com][9], [lwn.net][10])
\[9] EAS: выбор CPU с учётом энергомодели. ([community.arm.com][11], [lkml.iu.edu][12])
\[10] Capacity-aware scheduling и критерий «capacity fitness». ([infradead.org][13])
\[11] Runtime PM подробности для драйверов. ([lkml.org][7])
\[12] Обзор EAS и big.LITTLE. ([static.lwn.net][2])
\[13] `uclamp` — документация ядра. ([docs.kernel.org][14], [Ядро Linux][15])
\[14] uclamp в пользовательском пространстве (`uclampset(1)`). ([man7.org][16])
\[15] cpusets — ограничение размещения задач. ([docs.kernel.org][17])
\[16] Cgroup v2 — общая справка. ([docs.kernel.org][18])

[1]: https://docs.kernel.org/6.2/scheduler/sched-capacity.html?utm_source=chatgpt.com "Capacity Aware Scheduling"
[2]: https://static.lwn.net/kerneldoc/scheduler/index.html?utm_source=chatgpt.com "Scheduler — The Linux Kernel documentation"
[3]: https://review.lineageos.org/362451?utm_source=chatgpt.com "cpufreq: schedutil: Fix iowait boost reset - Gerrit Code Review"
[4]: https://linux.kernel.narkive.com/5fwSvO2K/patch-2-2-sched-use-iowait-boost-policy-option-in-schedutil?utm_source=chatgpt.com "[PATCH 2/2] sched: Use iowait boost policy option in schedutil"
[5]: https://docs.kernel.org/power/freezing-of-tasks.html?utm_source=chatgpt.com "Freezing of tasks - The Linux Kernel documentation"
[6]: https://www.infradead.org/~mchehab/kernel_docs/admin-guide/pm/suspend-flows.html?utm_source=chatgpt.com "System Suspend Code Flows — The Linux Kernel 5.10.0-rc1+ ..."
[7]: https://lkml.org/lkml/2017/7/16/90?utm_source=chatgpt.com "LKML: Joel Fernandes: [PATCH RFC v5] cpufreq: schedutil: Make iowait ..."
[8]: https://docs.kernel.org/admin-guide/pm/sleep-states.html?utm_source=chatgpt.com "System Sleep States"
[9]: https://android.googlesource.com/kernel/common.git/%2B/refs/heads/mirror-pa-android12-5.10-staging/Documentation/ABI/testing/sysfs-power?utm_source=chatgpt.com "Documentation/ABI/testing/sysfs-power - kernel/common.git"
[10]: https://lwn.net/Articles/479841/?utm_source=chatgpt.com "Autosleep and wake locks"
[11]: https://community.arm.com/arm-community-blogs/b/architectures-and-processors-blog/posts/energy-aware-scheduling-in-linux?utm_source=chatgpt.com "Energy Aware Scheduling (EAS) in Linux 5.0"
[12]: https://lkml.iu.edu/1812.0/00841.html?utm_source=chatgpt.com "[PATCH v10 00/15] Energy Aware Scheduling"
[13]: https://www.infradead.org/~mchehab/kernel_docs/scheduler/sched-capacity.html?utm_source=chatgpt.com "Capacity Aware Scheduling — The Linux Kernel 5.10.0-rc1+ ..."
[14]: https://docs.kernel.org/scheduler/sched-util-clamp.html?utm_source=chatgpt.com "Utilization Clamping - The Linux Kernel documentation"
[15]: https://kernel.org/doc/html/latest/scheduler/sched-util-clamp.html?highlight=util+clamp&utm_source=chatgpt.com "Utilization Clamping — The Linux Kernel documentation"
[16]: https://man7.org/linux/man-pages/man1/uclampset.1.html?utm_source=chatgpt.com "uclampset(1) - Linux manual page"
[17]: https://docs.kernel.org/admin-guide/cgroup-v1/cpusets.html?utm_source=chatgpt.com "CPUSETS"
[18]: https://docs.kernel.org/admin-guide/cgroup-v2.html?utm_source=chatgpt.com "Control Group v2 — The Linux Kernel documentation"
