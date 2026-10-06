# reactingFoamUnitLe

Солвер для расчёта горения с химическими реакциями на базе стандартного
`reactingFoam` из OpenFOAM v2606, с **единственным** отличием:

> **Число Льюиса задано равным единице (Le = 1)** для диффузии химических
> компонентов.

## Зачем это нужно

В стандартном `reactingFoam` диффузионный член уравнения переноса массовых
долей записан через динамическую вязкость:

```cpp
- fvm::laplacian(turbulence->muEff(), Yi)
```

Это даёт эффективный коэффициент диффузии компонента

$$D = \frac{\mu}{\rho} = \nu,$$

то есть фактически задаётся **число Шмидта $Sc = 1$**, а не $Le = 1$.
При этом число Льюиса получается

$$Le = \frac{\alpha}{D} = \frac{\kappa/(\rho C_p)}{\mu/\rho}
     = \frac{\kappa}{\mu C_p} = \frac{1}{Pr} \approx \frac{1}{0.7} \approx 1.43.$$

Для механизма `2S_CH4_BFER` (Franzelli et al., *Combustion and Flame* 159,
2012, 621–637) в оригинальной статье принято допущение **единичного числа
Льюиса** ($Le = 1$). Чтобы расчёт соответствовал статье, диффузия массы должна
идти с тем же коэффициентом, что и диффузия тепла:

$$D = \alpha = \frac{\kappa}{\rho C_p}.$$

## Что изменено относительно стандартного reactingFoam

Единственное изменение — в файле `YEqn.H`:

```cpp
// Было (стандартный reactingFoam, Le = 1/Pr ≈ 1.43):
- fvm::laplacian(turbulence->muEff(), Yi)

// Стало (Le = 1):
- fvm::laplacian(thermo.alpha(), Yi)
```

Здесь `thermo.alpha()` — это ламинарная тепловая диффузия энтальпии
$\kappa/C_p$ в единицах `[kg/m/s]`, что ровно соответствует $\rho D$ при
$Le = 1$.

Дополнительно в `Make/options` исправлена нестандартная переменная
`$(LIB_SRC)` → `$(FOAM_SRC)` (иначе include-пути не собираются в стандартном
окружении OpenFOAM).

## Сборка

```bash
# 1. Подключить окружение OpenFOAM
source /usr/lib/openfoam/openfoam2606/etc/bashrc

# 2. Перейти в папку солвера
cd reactingFoamUnitLe

# 3. Очистить предыдущую сборку (если была)
wclean

# 4. Собрать
wmake
```

После сборки исполняемый файл появится в:

```
$FOAM_USER_APPBIN/reactingFoamUnitLe
```

## Запуск

В `system/controlDict` вашего кейса укажите:

```
application     reactingFoamUnitLe;
```

и подключите нужные библиотеки (например, для механизма `2S_CH4_BFER`):

```
libs
(
    "$FOAM_USER_LIBBIN/libtwoSCH4Bfer.so"
    "$FOAM_USER_LIBBIN/libfixedValueEquivalenceRatioFvPatchFields.so"
);
```

Затем запускайте как обычный `reactingFoam`:

```bash
# последовательно
reactingFoamUnitLe

# или параллельно (пример на 5 процессах)
mpirun -np 5 reactingFoamUnitLe -parallel
```

## Требования

- OpenFOAM v2606 (собирался и тестировался на этой версии).
- Компилятор GCC (стандартное окружение OpenFOAM).

## Лицензия

Код основан на `reactingFoam` из OpenFOAM, который распространяется под
лицензией **GNU GPL v3**. Соответственно, данный солвер также распространяется
под **GPL v3**.

## Ссылки

- Franzelli B., Riber E., Sanjosé M., Poinsot T.
  *A two-step chemical scheme for kerosene–air premixed flames*,
  Combustion and Flame 159 (2012) 621–637.
  (механизм `2S_CH4_BFER`, допущение $Le = 1$)
- OpenFOAM: https://www.openfoam.com