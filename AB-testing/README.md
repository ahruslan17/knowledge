## A/B‑тестирование — оглавление конспектов (ipynb)

Ниже — структурированное содержание цикла ноутбуков. Ссылки ведут на планируемые файлы в `A/B Testing/`. Материал идёт от основ к продвинутым техникам и инженерным аспектам платформы экспериментов.

### Навигация
- Нумерация тем: 00, 01, 02, ... — для последовательного прохождения.
- Формат: Jupyter‑ноутбуки с краткими объяснениями, формулами, графиками и практикумами.
- Начинать отсюда: 00 (обзор и как пользоваться материалом).

---

### 0. Введение
- [00_overview_plan.ipynb](A/B%20Testing/00_overview_plan.ipynb) — цели, структура, как читать и запускать ноутбуки

### 1. Основы экспериментов
- [01_experiment_fundamentals.ipynb](A/B%20Testing/01_experiment_fundamentals.ipynb) — что такое A/B, рандомизация, экспозиция, конфаундеры
- [01a_randomization_checks.ipynb](A/B%20Testing/01a_randomization_checks.ipynb) — A/A и проверки баланса групп

### 2. Метрики
- [02_metrics_design.ipynb](A/B%20Testing/02_metrics_design.ipynb) — выбор метрик: пропорции, rates, средние, квантили; свойства
- [02a_guardrail_metrics.ipynb](A/B%20Testing/02a_guardrail_metrics.ipynb) — guardrail‑метрики и иерархия целей

### 3. Дизайн и расчёт выборки
- [03_power_sample_size.ipynb](A/B%20Testing/03_power_sample_size.ipynb) — мощность, MDE, расчёт N (пропорции/средние)
- [03a_duration_exposure.ipynb](A/B%20Testing/03a_duration_exposure.ipynb) — длительность, сезонность, экспозиция, инкрементальность

### 4. Частотный анализ A/B
- [04_frequentist_tests.ipynb](A/B%20Testing/04_frequentist_tests.ipynb) — z/t‑тесты, пропорции (pooled/unpooled), Welch
- [04a_confidence_intervals.ipynb](A/B%20Testing/04a_confidence_intervals.ipynb) — доверительные интервалы и практическая значимость
- [04b_bootstrap_basics.ipynb](A/B%20Testing/04b_bootstrap_basics.ipynb) — бутстрэп для средних/медиан/квантилей

### 5. Последовательное тестирование
- [05_sequential_testing.ipynb](A/B%20Testing/05_sequential_testing.ipynb) — «подглядывание», alpha‑spending, O’Brien–Fleming
- [05a_sprt_and_group_sequential.ipynb](A/B%20Testing/05a_sprt_and_group_sequential.ipynb) — SPRT и group‑sequential дизайны

### 6. Множественные проверки
- [06_multiple_testing.ipynb](A/B%20Testing/06_multiple_testing.ipynb) — Bonferroni, Holm, BH/FDR, FWER

### 7. Непараметрические методы
- [07_nonparametric_tests.ipynb](A/B%20Testing/07_nonparametric_tests.ipynb) — Mann–Whitney, перестановочные тесты

### 8. Ковариаты и снижение дисперсии
- [08_covariates_cuped.ipynb](A/B%20Testing/08_covariates_cuped.ipynb) — ковариатные корректировки, CUPED/CUPAC
- [08a_stratification_matching.ipynb](A/B%20Testing/08a_stratification_matching.ipynb) — стратификация, матчинг, блок‑рандомизация

### 9. Байесовский анализ
- [09_bayesian_ab.ipynb](A/B%20Testing/09_bayesian_ab.ipynb) — бета‑биномиальная модель, постериоры, ROPE/HDI
- [09a_bayesian_means.ipynb](A/B%20Testing/09a_bayesian_means.ipynb) — Normal‑Normal, t‑подходы для средних
- [09b_bandits_overview.ipynb](A/B%20Testing/09b_bandits_overview.ipynb) — обзор multi‑armed bandits: UCB, Thompson

### 10. Гетерогенность эффектов и uplift
- [10_heterogeneity_uplift.ipynb](A/B%20Testing/10_heterogeneity_uplift.ipynb) — CATE, uplift‑модели, сегментация

### 11. Сложные дизайны
- [11_switchback_tests.ipynb](A/B%20Testing/11_switchback_tests.ipynb) — switchback (time‑based) для зависимых трафиков
- [11a_geo_experiments.ipynb](A/B%20Testing/11a_geo_experiments.ipynb) — GEO‑тесты, кластерные дизайны, синтетический контроль (обзор)
- [11b_factorial_mgt.ipynb](A/B%20Testing/11b_factorial_mgt.ipynb) — факторные дизайны и MGT

### 12. Особенности данных и метрик
- [12_censored_truncated_data.ipynb](A/B%20Testing/12_censored_truncated_data.ipynb) — цензурирование, трайкейшн, лаги конверсий
- [12a_revenue_heavy_tails.ipynb](A/B%20Testing/12a_revenue_heavy_tails.ipynb) — heavy‑tails выручки, winsorization/trim, robust‑методы
- [12b_retention_cohorts.ipynb](A/B%20Testing/12b_retention_cohorts.ipynb) — когорты, удержание, LTV, лаг‑структуры

### 13. Причинно‑следственный анализ (в контексте экспериментов)
- [13_causal_inference_overview.ipynb](A/B%20Testing/13_causal_inference_overview.ipynb) — потенц. исходы, ATE/ITT/ATT
- [13a_diff_in_diff.ipynb](A/B%20Testing/13a_diff_in_diff.ipynb) — Difference‑in‑Differences, event‑study
- [13b_propensity_ipw_matching.ipynb](A/B%20Testing/13b_propensity_ipw_matching.ipynb) — propensity score, IPW, матчинг
- [13c_instrumental_variables.ipynb](A/B%20Testing/13c_instrumental_variables.ipynb) — инструментальные переменные (обзор)

### 14. Инженерные аспекты и платформа
- [14_assignment_and_exposure.ipynb](A/B%20Testing/14_assignment_and_exposure.ipynb) — assignment, единицы рандомизации, бакетизация
- [14a_logging_data_quality.ipynb](A/B%20Testing/14a_logging_data_quality.ipynb) — логирование, идентификаторы, качество данных
- [14b_metrics_pipeline.ipynb](A/B%20Testing/14b_metrics_pipeline.ipynb) — пайплайн метрик, задержки, backfills
- [14c_experiment_service_design.ipynb](A/B%20Testing/14c_experiment_service_design.ipynb) — фичефлаги, конфиги, rollouts, guardrails

### 15. Надёжность и эксплуатация
- [15_aa_tests_and_validation.ipynb](A/B%20Testing/15_aa_tests_and_validation.ipynb) — A/A, дрейф, мониторинг и алёртинг
- [15a_common_pitfalls.ipynb](A/B%20Testing/15a_common_pitfalls.ipynb) — анти‑паттерны: p‑hacking, Simpson, сезонность
- [15b_checklists.ipynb](A/B%20Testing/15b_checklists.ipynb) — чек‑листы запуска, анализа и деактивации теста

### 16. Практикумы и симуляции
- [16_simulations_power_peeking.ipynb](A/B%20Testing/16_simulations_power_peeking.ipynb) — мощность, peeking, sequential (симуляции)
- [16a_bootstrap_playground.ipynb](A/B%20Testing/16a_bootstrap_playground.ipynb) — бутстрэп‑практикум
- [16b_metric_sensitivity.ipynb](A/B%20Testing/16b_metric_sensitivity.ipynb) — чувствительность метрик, MDE‑калькулятор

### 17. Кейсы и этика
- [17_product_cases.ipynb](A/B%20Testing/17_product_cases.ipynb) — end‑to‑end кейсы: гипотеза → решение
- [17a_tradeoffs_ethics.ipynb](A/B%20Testing/17a_tradeoffs_ethics.ipynb) — trade‑offs, риски, этика экспериментов

### 18. Справка
- [18_glossary_cheatsheet.ipynb](A/B%20Testing/18_glossary_cheatsheet.ipynb) — глоссарий и шпаргалка формул
- [18a_references.ipynb](A/B%20Testing/18a_references.ipynb) — ссылки на статьи, книги, доклады

---

Лицензия: см. [`LICENSE`](LICENSE).

