[Bike_Sharing_DES_Optimization_GitHub_Summary_v2.md](https://github.com/user-attachments/files/28522501/Bike_Sharing_DES_Optimization_GitHub_Summary_v2.md)
# Bike Sharing System Rebalancing Using Discrete-Event Simulation and Optimization

## Project Summary

This project studies the overnight rebalancing problem in a bike sharing system. Bike sharing systems provide a flexible and environmentally sustainable mode of urban transportation, but their performance depends heavily on whether bikes and empty docks are available when riders need them. If too few bikes are available at a station, customers who want to start a trip are lost. If too few empty docks are available, customers who want to end a trip must search for another station. These two failure modes create an imbalance problem that can reduce rider satisfaction and system reliability.

The goal of this project is to evaluate whether an optimization-based overnight rebalancing policy can improve system performance under uncertainty. The model combines a mathematical optimization formulation for deciding midnight bike allocations with a discrete-event simulation (DES) that represents day-to-day system operations. The simulation tracks rider arrivals, trip completions, station inventory levels, missed demand, station capacity, and bike reliability. The optimization model is then used as a control policy inside the simulation to compare a rebalanced system against an imbalanced baseline.

## Problem Setting

The system consists of five bike sharing stations with fixed dock capacities and an initial number of bikes at each station. Riders arrive at stations over time to rent bikes, travel to destination stations, and return bikes if an empty dock is available. The numerical setting uses time-dependent arrival rates to reproduce realistic demand patterns, including morning and evening rush-hour behavior. Ride times are exponentially distributed with station-pair-specific means, and origins and destinations are sampled across the five stations.

A central challenge is that the observed station inventory process does not fully reveal true demand. When a station is empty, a rider who wants to rent a bike is missed, and the station level remains zero. Similarly, when a station is full, a rider who wants to return a bike cannot dock there. Simply using observed inventory levels may therefore underestimate true demand. To address this, the optimization model is based on masked demand histories, leaving out periods where a station is empty or full and hence observed demand is censored.

## Optimization Model

The optimization model is the decision-making component of the project. At each midnight decision epoch, the system manager chooses how many bikes should be placed at each station before the next day begins. These allocations are target inventory levels: they are not routing decisions for a repositioning truck, but rather station-level targets that specify the desired number of bikes after overnight rebalancing.

### Notation

Let:

- `s` denote a bike sharing station.
- `X_s` denote the number of bikes allocated to station `s` overnight.
- `c(s)` denote the capacity of station `s`, i.e., the number of docks at the station.
- `B` denote the total number of bikes available for allocation.
- `f_t^s` denote the predicted net flow of bikes at station `s` at time `t`, where net flow is defined as cumulative demand in minus cumulative demand out.
- `D` denote the maximum imbalance across all stations.

The main decision variables are the allocation variables `X_s` and the auxiliary imbalance variable `D`. The allocation variables are integer-valued because a station can only receive a whole number of bikes. They are also bounded by station capacity, so `0 <= X_s <= c(s)`.

### Base formulation

The optimization problem can be written as:

```text
minimize        D

subject to      sum_s X_s <= B

                D >= X_s + min_t f_t^s                    for every station s

                D >= c(s) - X_s - max_t f_t^s              for every station s

                X_s in {0, 1, ..., c(s)}                   for every station s

                D >= 0
```

The first constraint limits the total number of bikes allocated across all stations. The model uses `<= B` rather than `= B` so that the optimization is not forced to place every bike into the system if doing so would make the station network more imbalanced.

The two imbalance constraints are the key part of the model. They evaluate the worst projected condition of each station over the next day. Starting the day with `X_s` bikes, the projected number of bikes at station `s` after the net flow up to time `t` is approximately `X_s + f_t^s`. The term `min_t f_t^s` therefore identifies the most bike-depleting time of the day for station `s`. The constraint

```text
D >= X_s + min_t f_t^s
```

controls the station's worst bike availability. If this value is close to zero, the station is at risk of becoming empty, which creates missed outgoing demand.

The second imbalance constraint focuses on the other side of service quality: empty docks. If the projected inventory is `X_s + f_t^s`, then the projected number of empty docks is `c(s) - X_s - f_t^s`. The smallest number of empty docks occurs when the net inflow is largest, which is why the formulation uses `max_t f_t^s`. The constraint

```text
D >= c(s) - X_s - max_t f_t^s
```

controls the station's worst dock availability. If this value is close to zero, the station is at risk of becoming full, which creates missed incoming demand because riders cannot return bikes there.

By minimizing `D`, the model seeks an allocation that balances these two risks across all stations. In practical terms, the optimizer tries to avoid putting too few bikes at high-demand origin stations while also avoiding overfilling stations that are likely to receive many returns during the day. The formulation is compact, but it captures the central operational tradeoff in bike sharing: every extra bike placed at one station improves bike availability there, but it also consumes a dock and removes a bike from potential use elsewhere.

### Demand estimation and masking

The inputs `f_t^s` are estimated from historical or simulated demand information. A major issue is that observed station levels are censored when stations are empty or full. For example, if a station has no bikes, additional customers who want to rent a bike are not observed through inventory changes. Similarly, if a station is full, additional attempted returns do not increase the observed station level.

To reduce this bias, the project uses masked demand histories. Periods in which a station is empty or full are excluded when estimating net flow. This helps the optimization model use a more realistic estimate of underlying demand rather than treating stockouts or full-station events as if demand had disappeared.

### Chance-constrained bike availability

The project also extends the total-bike constraint to account for maintenance-related uncertainty. Not every physical bike is assumed to be usable: each bike is reliable with probability `p`, so the total number of usable bikes is random. If the number of usable bikes is modeled as a binomial random variable, then a deterministic allocation of all nominal bikes may be infeasible when some bikes are broken.

To handle this, the model replaces the deterministic total-bike constraint with a chance-constrained version. Instead of requiring allocations to be below the nominal number of bikes, the allocation total is bounded by a conservative quantile of the usable-bike distribution:

```text
sum_s X_s <= F^{-1}(epsilon) + 1
```

where `F^{-1}(epsilon)` is a quantile of the bike-availability distribution. This makes the allocation feasible with the desired reliability level and prevents the policy from assuming that all bikes will always be usable. In the simulation, bike failures are realized at midnight after allocation decisions are made, and the balanced system removes broken bikes from stations during the repositioning process.

### Role of the optimization model in the simulation

The optimization model is not solved once in isolation. It is embedded inside the discrete-event simulation as a repeated midnight control policy. At each intervention day, the simulation uses the available masked demand history to estimate station-level net flows, solves the allocation model, and then updates station inventories according to the optimal target vector. This simulation-optimization loop allows the project to evaluate not only whether the mathematical allocation is reasonable, but also whether the policy improves realized system performance under stochastic rider arrivals, trip durations, station congestion, and bike reliability uncertainty.

The Python implementation solves the integer optimization model using Gurobi.

## Discrete-Event Simulation

The simulation model represents the evolution of the bike sharing system over time. It uses three main event types:

1. **Arrival event**: A rider arrives at a station to rent a bike. If a bike is available, the bike is removed from the station and a future trip-completion event is scheduled. If no bike is available, an outgoing demand is missed.
2. **Departure event**: A rider arrives at a destination station to return a bike. If an empty dock is available, the bike is returned. If the station is full, an incoming demand is missed and the rider is routed to another station.
3. **Midnight event**: The system updates daily performance statistics, realizes bike reliability, and, in the intervention phase, applies the optimization model to rebalance bikes across stations.

The simulation tracks served trips, missed outgoing demand, missed incoming demand, total missed demand, station levels, reliability indicators, and daily allocation decisions. To create a fair comparison, each replication is run twice with the same random seed: once as an imbalanced baseline without rebalancing, and once with the optimization policy activated after an initial as-is period.

## Numerical Experiment

The numerical experiment uses 100 replications over a 500-day simulation horizon. The first 250 days are treated as an as-is learning period, and the optimization policy is activated after that point. The model uses 46 initial bikes across five stations, station capacities ranging from 6 to 18 docks, time-varying exponential interarrival times, exponential ride times, and a baseline bike reliability probability of 0.8. The net flow inputs to the optimization model are estimated by averaging simulated masked demand histories.

The main performance measure is missed demand. The results show that the optimization-based rebalancing policy substantially reduces total missed demand compared with the imbalanced baseline. The improvement comes mainly from a large reduction in missed incoming demand, while the system accepts a small increase in missed outgoing demand. This tradeoff reflects the purpose of rebalancing: instead of overconcentrating bikes at some stations and leaving too few docks elsewhere, the optimized allocations improve overall system balance.

## Sensitivity Analysis

A sensitivity analysis is conducted over bike reliability values from 0.70 to 0.90. The results indicate that the value of rebalancing is especially important when bike reliability is lower and the number of usable bikes is more uncertain. As reliability improves, the gap between the balanced and imbalanced systems narrows for some performance measures, but the optimization policy continues to provide a structured way to allocate bikes under uncertainty.

## Code Structure

The notebook implements the full simulation-optimization pipeline in Python. The main components include:

- Input definitions for station capacities, initial bike levels, arrival rates, ride-time distributions, and reliability probabilities.
- Helper functions for exponential sampling, time-of-day classification, masked net-flow calculation, and bike reliability realization.
- A Gurobi optimization function that computes station-level bike allocations under capacity and chance-constrained total-bike availability.
- Event functions for rider arrivals, trip completions, and midnight rebalancing.
- A simulation loop based on a future event list.
- Replication logic for comparing balanced and imbalanced systems under matched random seeds.
- Visualization of missed demand outcomes and sensitivity analysis.

## Requirements

The project uses the following main Python packages:

- `numpy`
- `scipy`
- `matplotlib`
- `gurobipy`

A valid Gurobi installation and license are required to solve the optimization model.

## Key Takeaway

This project demonstrates how discrete-event simulation and mathematical optimization can be integrated to evaluate operational policies in shared mobility systems. The optimization model provides daily bike allocation decisions, while the simulation environment tests those decisions under stochastic rider demand, travel times, and bike availability uncertainty. The results suggest that even a relatively compact optimization model can substantially improve bike sharing system reliability when embedded in a realistic simulation framework.
