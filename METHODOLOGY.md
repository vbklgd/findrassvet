# findrassvet.ru methodology

[Русская версия](METHODOLOGY.ru.md)

How [findrassvet.ru](https://findrassvet.ru) turns NORAD catalog data into altitude, orbit-raising rate and coverage zones for the Rassvet constellation (Bureau 1440). Every numerical parameter below was derived from the element-set archive, not tuned by eye. The verification protocols live in the site code; their results are summarized here.

The site is unofficial and not affiliated with Bureau 1440 LLC.

## Contents

1. [Data](#1-data)
2. [Semi-major axis altitude](#2-semi-major-axis-altitude)
3. [Orbit-raising rate](#3-orbit-raising-rate)
4. [Coverage zone](#4-coverage-zone)
5. [Finding: plane separation by nodal precession](#5-finding-plane-separation-by-nodal-precession)
6. [How to cite](#6-how-to-cite)

## 1. Data

Orbital elements come from the public NORAD catalog. Only the raw element sets are stored; semi-major axis, perigee, apogee and rate are recomputed on every request.

## 2. Semi-major axis altitude

The mean motion in an element set is a Kozai mean motion: the secular effect of Earth oblateness is already mixed in. Computing $`a = (\mu/n^2)^{1/3}`$ directly gives a value offset from the semi-major axis SGP4 actually works with. The site therefore reproduces the Kozai → Brouwer step of SGP4 initialization (`sgp4init`, Spacetrack Report #3) with WGS-72 constants:

```math
\begin{aligned}
n_0 &= \frac{2\pi\,n}{1440} && \text{rad/min} \\
a_1 &= \left(\frac{k_e}{n_0}\right)^{2/3} && \text{Earth radii} \\
\delta_1 &= \frac{3}{4}\,\frac{J_2\,(3\cos^2 i - 1)}{a_1^2\,(1-e^2)^{3/2}} \\
a_0 &= a_1\left(1 - \frac{\delta_1}{3} - \delta_1^2 - \frac{134}{81}\,\delta_1^3\right) \\
\delta_0 &= \frac{3}{4}\,\frac{J_2\,(3\cos^2 i - 1)}{a_0^2\,(1-e^2)^{3/2}} \\
a &= \frac{a_0}{1-\delta_0}\,R_E && \text{Brouwer semi-major axis, km} \\
h &= a - R_E
\end{aligned}
```

$`\mu = 398600.8`$ km³/s², $`R_E = 6378.135`$ km, $`J_2 = 0.001082616`$, $`k_e = 60/\sqrt{R_E^3/\mu}`$.

Offset of the "Keplerian" semi-major axis relative to Brouwer (e = 2·10⁻⁴):

| Inclination | h = 300 km | h = 508 km | h = 800 km |
|---|---|---|---|
| 51.6° | −0.52 km | −0.50 km | −0.48 km |
| 82.3° | +3.12 km | +3.03 km | +2.90 km |
| 97.4° | +3.13 km | +3.04 km | +2.91 km |

The offset is nearly constant and does not change the shape of the orbit-raising curve, but three kilometres show on the absolute axis label. The implementation agrees with satellite.js to within 10⁻⁹ km.

Perigee and apogee: $`a\,(1 \mp e) - R_E`$.

Semi-major axis altitude is not the flight altitude: the instantaneous radius oscillates around it by ±a·e over one revolution. At e ≈ 2·10⁻⁴ that is ±1.4 km; on a transfer orbit after a burn, tens of kilometres. Orbit raising is only cleanly visible in the semi-major axis, which is why it is the primary quantity.

## 3. Orbit-raising rate

Rate is the time derivative of semi-major axis altitude, km/day. On the site it is the RATE (ТЕМП) quantity on the chart and the column in the table; it was added to the chart on 29 August 2026.

### 3.1. Data properties

Archive as of 2026-08-28 (31 satellites):

| Quantity | Value |
|---|---|
| Interval between element sets | median 7.5 h; 10 % shorter than 1.5 h, 10 % longer than 18.1 h |
| Element sets per day | 2.9 for the March launch, 1.7 for the July launch |
| Single-set fit noise σ | 0.166 km (RMS residual from a line on passive arcs, 2393 points) |
| Engine burns | 147 segments; median 2.0 days, quartiles 1.0 and 4.5 days; one third shorter than two days |

The true rate is a staircase: passive arcs (atmospheric drag, about −0.05 km/day) alternate with burns.

### 3.2. Why not differences of adjacent sets

Differencing adjacent sets gives rate noise $`\sigma\sqrt{2}/\Delta t \approx 0.75`$ km/day at the median interval, enough to make passive flight look like thrusting. Rejected.

### 3.3. Adopted algorithm

For each calendar day d (UTC), fit an ordinary least-squares line to all element sets in a 2-day window centred on that day's noon, [d + 0.5 − 1, d + 0.5 + 1]. The day's rate is the slope. No point is produced if:

- the window holds fewer than 2 sets;
- the epochs in the window span less than 0.8 day;
- all sets lie on one side of noon.

The last rule forbids extrapolation: without it, the fresh end of the chart disagreed with the table column by 5 km/day.

### 3.4. Window width

A window narrower than two days loses days to sparse publication; a wider one smears short burns (a third of them are shorter than two days).

| Window, days | Min. sets | Min. span, days | Days covered |
|---|---|---|---|
| 1.5 | 2 | 0.6 | 74 % |
| 1.5 | 3 | 0.6 | 69 % |
| **2.0** | **2** | **0.8** | **87 %** |
| 2.0 | 3 | 0.8 | 84 % |
| 2.5 | 2 | 1.0 | 92 % |
| 3.0 | 2 | 1.2 | 95 % |

### 3.5. Performance

- Estimate noise 0.11 km/day against a typical thrusting rate of 4–5 km/day.
- A one-day burn keeps 68 % of its peak in the estimate, a two-day burn 85 % (measured on synthetic data with a known answer on the real epoch grid).
- The "thrusting" threshold of 0.8 km/day is about five σ of the estimate; no false triggers.
- On the archive: 1920 points, rate from −10.9 to +14.8 km/day; 54 % of days within ±0.5 km/day (passive), 31 % above +1 km/day, 1 % below −1 km/day.

### 3.6. Limitations

The time integral of the rate falls 10–12 % short of the actual altitude gain: a burn that falls entirely inside a publication gap cannot be recovered. The chart answers "when and how fast", not "how many kilometres in total".

### 3.7. Cross-check

Series extracted from the in-browser chart were compared with an independent Python implementation: 1928 common points across 32 satellites, maximum difference 5.2·10⁻¹¹ km/day, no missing or extra points.

## 4. Coverage zone

The zone boundary is the set of surface points from which the satellite is seen at elevation ε. On a spherical Earth this is a small circle with central angle

```math
\lambda = \arccos\left(\frac{R_E}{R_E+h}\,\cos\varepsilon\right) - \varepsilon
```

Checks: the formula was derived two independent ways (via the nadir angle); every point of the generated circle reproduces ε to within 4·10⁻¹² degrees; limiting cases and reference values agree (ISS at 400 km: 2200 km radius; GEO: 81.3°).

Cost of the spherical assumption on the WGS-84 ellipsoid: about 0.1° in elevation and up to 7.5 km in boundary position.

Zone radius on the ground, km:

| h, km | ε = 0° | ε = 5° | ε = 10° | ε = 25° |
|---|---|---|---|---|
| 300 | 1917 | 1437 | 1096 | 561 |
| 400 | 2200 | 1710 | 1343 | 720 |
| 517 | 2483 | 1985 | 1597 | 893 |

When the circle covers a pole ($`90^\circ - |\varphi| < \lambda`$), its latitude–longitude outline is not closed, and the map draws it through the pole, extended to the map edge.

## 5. Finding: plane separation by nodal precession

By late August 2026 the right ascensions of the ascending node of the March-launch satellites had spread by 7.7°. The spread is explained to within 0.02° (correlation 1.0000) by J₂ secular nodal regression,

```math
\dot{\Omega} = -\frac{3}{2}\,n\,J_2\left(\frac{R_E}{p}\right)^2\cos i, \qquad p = a\,(1-e^2)
```

integrated over each satellite's own altitude history. Three satellites that stayed for months at 348–429 km instead of the operational 508 km accumulated 4.4–4.9° more regression than the rest. In other words, the staircase on the altitude chart is the constellation being spread into planes. The July launch was still a single plane at that time (0.7° spread); the two launches' planes are 113° apart.

## 6. How to cite

The methodology and charts may be used freely with attribution: "findrassvet.ru, V. Bulybenko". For academic work, use the "Cite this repository" button at the top right (file [CITATION.cff](CITATION.cff)).
