# Synthetic Tunnel Load Generator

Synthetic electrical load generator for road tunnels, implemented as a Streamlit application.

The tool generates a configurable electrical consumption time series for a road tunnel, typically at a 10-minute time step over one or several years. It combines a deterministic engineering model with a Monte Carlo stochastic layer. The model explicitly separates lighting demand, ventilation demand, auxiliary systems, traffic-driven effects, pollution peaks, accident events and random variability.

The objective is to provide a transparent, parametric and scientifically interpretable generator of synthetic tunnel electricity demand. The model is not calibrated by default on a specific tunnel. Engineering use requires calibration against measured consumption, tunnel geometry, traffic data, ventilation rules and lighting characteristics.

---

## 1. Scientific objective

Road tunnels are energy-intensive infrastructures. Their electricity consumption is mainly driven by:

- permanent safety and monitoring systems;
- lighting requirements;
- ventilation requirements;
- traffic intensity;
- tunnel geometry;
- operating rules;
- external and accidental events such as congestion, pollution peaks or incidents.

This application provides a synthetic electrical-load generator designed for:

- exploratory simulation;
- sensitivity analysis;
- teaching and demonstration;
- generation of synthetic benchmark datasets;
- evaluation of load-shape variability;
- preliminary assessment of the influence of tunnel geometry and operating conditions on electricity demand.

The tool focuses only on electrical load generation. It does not include photovoltaic production, wind generation, battery storage, energy management or grid optimization.

---

## 2. Model overview

The total electrical power demand is decomposed as:

```math
P(t) = P_{light}(t) + P_{vent}(t) + P_{aux}(t) + \varepsilon(t)
```

where:

- `P_light(t)` is the lighting power demand;
- `P_vent(t)` is the ventilation power demand;
- `P_aux(t)` is the auxiliary electrical demand;
- `ε(t)` is a stochastic perturbation.

The corresponding energy over one time step is:

```math
E(t) = P(t) \Delta t
```

with:

```math
\Delta t = \frac{\Delta t_{min}}{60}
```

where `Δt_min` is the time step in minutes. For the default 10-minute resolution:

```math
\Delta t = \frac{10}{60} = \frac{1}{6} \text{ h}
```

The output energy is therefore expressed as:

```math
E_{kWh}(t) = P_{kW}(t) \times \frac{\Delta t_{min}}{60}
```

---

## 3. Temporal resolution

The simulator creates a regular time index:

```math
t_1, t_2, ..., t_N
```

with:

```math
N = \frac{D \times 24 \times 60}{\Delta t_{min}}
```

where:

- `D` is the number of simulated days;
- `Δt_min` is the time step in minutes.

For a two-year simulation with a 10-minute time step:

```math
N = \frac{730 \times 24 \times 60}{10} = 105120
```

The generated file therefore contains 105,120 rows for two years at 10-minute resolution.

---

## 4. Tunnel geometry

The model includes the following geometric and operational parameters:

| Symbol | Meaning |
|---|---|
| `L` | tunnel length in km |
| `L_m` | tunnel length in m |
| `N_tube` | number of tubes |
| `N_lane/tube` | number of lanes per tube |
| `N_lane` | total number of lanes |
| `H` | altitude in m |
| `D_max` | maximum depth or overburden proxy in m |
| `s` | mean absolute gradient in % |

The total number of lanes is:

```math
N_{lane} = N_{tube} \times N_{lane/tube}
```

Geometry affects the main load components:

```math
P_{light} \propto L \times N_{lane}
```

```math
P_{vent} \propto L \times N_{tube}
```

```math
P_{aux} \propto L \times N_{tube}
```

---

## 5. Seasonal structure

Each timestamp is assigned to one of four seasons:

| Season | Months |
|---|---|
| Winter | December, January, February |
| Spring | March, April, May |
| Summer | June, July, August |
| Autumn | September, October, November |

A seasonal correction factor is applied to represent the fact that tunnel demand may differ across the year due to daylight duration, environmental conditions and operational practices:

```math
f_{season}(t) \in \{f_{winter}, f_{spring}, f_{summer}, f_{autumn}\}
```

Default values used in the current implementation are:

```math
f_{winter}=1.12
```

```math
f_{spring}=0.98
```

```math
f_{summer}=0.92
```

```math
f_{autumn}=1.03
```

These values are first-order assumptions. They should be calibrated when real tunnel data are available.

---

## 6. Traffic model

Traffic is represented by a dimensionless index:

```math
T(t) \in [0, 1.5]
```

The daily traffic structure combines:

- a night baseline;
- a morning peak;
- an evening peak;
- a weekday/weekend correction;
- a seasonal correction;
- a tunnel-context correction;
- a daily random multiplier.

The morning and evening peaks are represented by Gaussian functions:

```math
G_m(t) = A_m \exp\left[-\frac{1}{2}\left(\frac{h(t)-h_m}{\sigma_m}\right)^2\right]
```

```math
G_e(t) = A_e \exp\left[-\frac{1}{2}\left(\frac{h(t)-h_e}{\sigma_e}\right)^2\right]
```

where:

- `h(t)` is the decimal hour of the day;
- `h_m` is the morning peak hour;
- `h_e` is the evening peak hour;
- `σ_m` and `σ_e` control the peak widths;
- `A_m` and `A_e` control the peak amplitudes.

The raw traffic signal is:

```math
T_{raw}(t) = T_{night}(t) + G_m(t) + G_e(t)
```

A daily random multiplier is then applied:

```math
T^*(t) = T_{raw}(t) f_{day}(t) f_{season}(t) f_{context} \eta_d
```

where:

```math
\eta_d \sim \mathcal{N}(1, \sigma_d^2)
```

Finally, the traffic index is normalized by a high quantile:

```math
T(t) = \frac{T^*(t)}{Q_{0.98}(T^*)}
```

and clipped to avoid unrealistic extremes:

```math
T(t) = \min\left(1.5, \max(0, T(t))\right)
```

---

## 7. Lighting model

The lighting demand depends on:

- tunnel length;
- number of lanes;
- lighting technology;
- daylight proxy;
- traffic index;
- minimum safety lighting level.

The installed lighting scale is:

```math
P_{light,scale} = p_{light}^{spec} \times L \times N_{lane}
```

where:

- `p_light^spec` is a technology-dependent specific lighting power in kW/km/lane;
- `L` is the tunnel length in km;
- `N_lane` is the total number of lanes.

The lighting power is then modeled as:

```math
P_{light}(t) = P_{light,scale}
\left[
f_{min}
+
(1-f_{min})(1-DL(t))
+
\beta_{light}T(t)
\right]
```

where:

- `DL(t)` is a daylight proxy between 0 and 1;
- `f_min` is the minimum lighting fraction;
- `β_light` is the traffic-lighting coupling coefficient;
- `T(t)` is the traffic index.

The daylight proxy is not an astronomical model. It is a simplified seasonal signal used to represent longer days in summer and shorter days in winter.

The current technology classes are:

| Lighting type | Interpretation |
|---|---|
| LED adaptive | lower demand, traffic-sensitive regulation |
| LED fixed | lower demand, weak traffic dependence |
| mixed | intermediate case |
| sodium fixed | older and more energy-intensive lighting |

---

## 8. Ventilation model

Ventilation demand depends on:

- tunnel length;
- number of tubes;
- ventilation technology;
- traffic intensity;
- altitude;
- slope;
- pollution events;
- accident events.

The ventilation scale is:

```math
P_{vent,scale} = p_{vent}^{spec} \times L \times N_{tube}
```

where:

- `p_vent^spec` is the specific ventilation power in kW/km/tube;
- `L` is the tunnel length in km;
- `N_tube` is the number of tubes.

The baseline ventilation demand is:

```math
P_{vent,0}(t)
=
P_{vent,scale}
\left(
f_{vent,min}
+
\alpha_T T(t)
\right)
f_{alt}
f_{slope}
```

where:

- `f_vent,min` is the minimum ventilation fraction;
- `α_T` is the traffic sensitivity;
- `T(t)` is the traffic index;
- `f_alt` is the altitude correction;
- `f_slope` is the slope correction.

The altitude factor is modeled as:

```math
f_{alt} = 1 + \frac{\max(H-500,0)}{6000}
```

The slope factor is modeled as:

```math
f_{slope} = 1 + \frac{|s|}{20}
```

These correction factors are deliberately simple. They provide first-order sensitivity but must not be interpreted as detailed fluid-dynamic modeling.

The ventilation technologies currently represented are:

| Ventilation type | Relative energy level |
|---|---|
| natural / low ventilation | low |
| longitudinal | moderate |
| semi-transverse | high |
| transverse | very high |

---

## 9. Pollution and accident events

Two stochastic binary event processes are included:

```math
I_{pollution}(t) \in \{0,1\}
```

```math
I_{accident}(t) \in \{0,1\}
```

For each simulated day, a Bernoulli trial determines whether a pollution or accident event occurs.

For pollution:

```math
B_{pollution,d} \sim \mathcal{B}(p_{pollution})
```

For accident:

```math
B_{accident,d} \sim \mathcal{B}(p_{accident})
```

If an event occurs, a random start hour and random duration are generated.

The ventilation demand is then amplified as:

```math
P_{vent}(t)
=
P_{vent,0}(t)
\left[
1
+
\alpha_{pollution} I_{pollution}(t)
+
\alpha_{accident} I_{accident}(t)
\right]
```

where:

- `α_pollution` is the pollution-event multiplier;
- `α_accident` is the accident-event multiplier.

This represents the operational increase in ventilation demand during degraded air-quality conditions, congestion or safety-related events.

---

## 10. Auxiliary load model

Auxiliary loads include:

- cameras;
- sensors;
- safety systems;
- IT systems;
- control systems;
- signaling;
- pumping;
- monitoring and communication systems.

The auxiliary power is modeled as:

```math
P_{aux}(t)
=
p_{aux}^{spec}
\times L
\times N_{tube}
\times f_{depth}
\times \xi(t)
```

where:

- `p_aux^spec` is the auxiliary specific load in kW/km/tube;
- `L` is tunnel length in km;
- `N_tube` is the number of tubes;
- `f_depth` is a depth or overburden correction;
- `ξ(t)` is a small stochastic multiplier.

The depth factor is:

```math
f_{depth} = 1 + \frac{\min(D_{max},800)}{6000}
```

The small stochastic multiplier is:

```math
\xi(t) \sim \mathcal{N}(1, \sigma_{aux}^2)
```

---

## 11. Monte Carlo structure

The simulator is stochastic. Each run corresponds to one Monte Carlo realization.

Randomness enters the model through:

1. daily traffic variability;
2. Gaussian multiplicative load noise;
3. pollution-event occurrence;
4. accident-event occurrence;
5. event start time;
6. event duration;
7. auxiliary load fluctuations.

The deterministic load is first computed:

```math
P_{det}(t)
=
P_{light}(t)
+
P_{vent}(t)
+
P_{aux}(t)
```

Then multiplicative Gaussian noise is applied:

```math
P(t)
=
P_{det}(t)
\left[
1 + \varepsilon(t)
\right]
```

with:

```math
\varepsilon(t) \sim \mathcal{N}(0, \sigma^2)
```

Finally, negative values are prevented:

```math
P(t) = \max(P(t),0)
```

The Monte Carlo seed controls reproducibility. Using the same seed produces the same synthetic time series. Changing the seed produces another plausible realization.

---

## 12. Inputs

The main user inputs are grouped into five categories.

### Time horizon

| Input | Unit | Description |
|---|---:|---|
| Start date | date | first timestamp |
| Number of days | days | simulation duration |
| Time step | minutes | temporal resolution |
| Monte Carlo seed | - | random seed |

### Tunnel geometry

| Input | Unit | Description |
|---|---:|---|
| Tunnel length | m | total length |
| Number of tubes | - | tunnel structure |
| Lanes per tube | - | traffic capacity |
| Altitude | m | environmental correction |
| Maximum depth / overburden proxy | m | proxy for auxiliary systems |
| Mean gradient | % | slope correction |

### Operating context

| Input | Description |
|---|---|
| Urban / peri-urban / rural | modifies traffic profile |
| Lighting type | modifies lighting power and control |
| Ventilation type | modifies ventilation scale |
| Auxiliary load | specific auxiliary power |
| Fixed non-scaled load | residual permanent load |

### Traffic structure

| Input | Description |
|---|---|
| Global traffic level | global multiplier |
| Morning peak hour | location of morning peak |
| Evening peak hour | location of evening peak |
| Peak width | peak duration |
| Ventilation traffic sensitivity | coupling between traffic and ventilation |

### Stochastic effects

| Input | Description |
|---|---|
| Gaussian noise sigma | unexplained variability |
| Pollution-event probability | daily probability of pollution peak |
| Accident probability | daily probability of accident |
| Pollution ventilation multiplier | ventilation increase during pollution |
| Accident ventilation multiplier | ventilation increase during accident |

---

## 13. Outputs

The downloaded CSV file contains one row per time step.

| Column | Description |
|---|---|
| timestamp | date and time |
| season | winter, spring, summer or autumn |
| day_type | weekday or weekend |
| length_m | tunnel length |
| n_tubes | number of tubes |
| n_lanes_total | total number of lanes |
| altitude_m | altitude |
| max_depth_m | maximum depth or overburden proxy |
| gradient_percent | mean absolute gradient |
| tunnel_context | urban, peri-urban or rural |
| lighting_type | lighting technology |
| ventilation_type | ventilation technology |
| traffic_index | normalized synthetic traffic index |
| pollution_event | binary pollution-event flag |
| accident_event | binary accident-event flag |
| lighting_kw | lighting power |
| ventilation_kw | ventilation power |
| auxiliary_kw | auxiliary power |
| power_kW | total electrical power |
| energy_kWh | electrical energy over the time step |

---

## 14. Plots generated by the application

The Streamlit interface displays several diagnostic plots.

### Total load decomposition

The main time-series plot shows:

- total electrical load;
- lighting load;
- ventilation load;
- auxiliary load.

This plot is used to verify the temporal behavior of the synthetic load and the relative contribution of each component.

### Seasonal mean daily profiles

The application computes the mean daily profile for each season:

```math
\bar{P}_{season}(h)
=
\frac{1}{N_{season,h}}
\sum_{t \in season, h(t)=h}
P(t)
```

This produces four average daily profiles:

- winter;
- spring;
- summer;
- autumn.

These profiles help verify whether the seasonal structure is coherent and whether morning/evening peaks are visible.

### Data preview

The application also displays the first rows of the generated dataset to allow quick inspection before downloading the CSV file.

---

## 15. Key performance indicators

The interface reports several indicators.

### Total simulated energy

```math
E_{tot} = \sum_t E(t)
```

### Annualized energy

```math
E_{year}
=
E_{tot}
\times
\frac{365}{D}
```

where `D` is the number of simulated days.

### Specific annual energy per meter

```math
E_{spec}
=
\frac{E_{year}}{L_m}
```

where `L_m` is the tunnel length in meters.

### Mean power

```math
\bar{P}
=
\frac{1}{N}
\sum_t P(t)
```

### Peak power

```math
P_{max}
=
\max_t P(t)
```

### Load factor

```math
LF
=
\frac{\bar{P}}{P_{max}}
```

These indicators provide first-order consistency checks.

---

## 16. Interpretation and scientific status

This simulator is a first-order synthetic model.

It is:

- transparent;
- parametric;
- reproducible;
- explainable;
- suitable for sensitivity analysis;
- suitable for educational and demonstrator purposes.

It is not, by default:

- a calibrated tunnel model;
- a regulatory engineering tool;
- a CFD ventilation model;
- a replacement for measured tunnel consumption;
- a validated operational digital twin.

For scientific or engineering use, the model should be calibrated against:

- measured electrical consumption;
- traffic counts;
- lighting specifications;
- ventilation operating rules;
- tunnel geometry;
- incident logs;
- meteorological or air-quality data.

---

## 17. Scope of the current version

The current version is limited to synthetic electrical-load generation for road tunnels.

It includes:

- tunnel geometry;
- lighting demand;
- ventilation demand;
- auxiliary electrical demand;
- seasonal effects;
- weekday/weekend effects;
- morning and evening traffic peaks;
- stochastic traffic variability;
- pollution events;
- accident events;
- Gaussian load perturbations;
- CSV export.

It does not include:

- photovoltaic production;
- wind generation;
- battery storage;
- grid import/export;
- cost optimization;
- carbon-emission accounting;
- control strategy or energy management.

These elements are intentionally excluded from the present version in order to keep the model focused, transparent and scientifically interpretable.

---

## 18. Running the application locally

Install dependencies:

```bash
pip install -r requirements.txt
```

Run the app:

```bash
streamlit run app.py
```

---

## 19. Deployment

The app can be deployed using Streamlit Community Cloud.

Required files:

```text
app.py
requirements.txt
README.md
LICENSE
```

The main application file is:

```text
app.py
```

---

## 20. License

MIT License.

---

## 21. Citation

If this tool is used in academic work, please cite the GitHub repository and specify the version or commit hash used to generate the synthetic data.

A formal citation file may be added later using `CITATION.cff`.
