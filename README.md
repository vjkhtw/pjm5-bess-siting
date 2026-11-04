# **Battery Siting and Sizing on the PJM 5-Bus Network**

Where should a battery go on a congested network, and how large should it be? This project answers that with a 24-hour DC optimal power flow (DC-OPF) on the PJM 5-bus test case, built in Python with Pyomo and solved with HiGHS. A battery is added at one candidate bus at a time, and daily operating-cost savings are compared across locations, power ratings and energy capacities. The model's dual variables (nodal prices and the Branch 6 shadow price) are then used to explain why the results come out as they do.

It builds on the single-period DC-OPF in [pjm5-opf](https://github.com/vjkhtw/pjm5-opf), which supplies the network, cost data and the validated base model.

## **Key results**

![][image1]

*Figure 1\. Daily operating-cost savings from a 50 MW / 200 MWh battery placed at each bus.*

*![][image2]*

*Figure 2\. Daily savings against energy capacity at each bus.*

*![][image3]*

*Figure 3\. Nodal prices at Bus 4 and Bus 5 (top) and the Branch 6 shadow price (bottom), with and without a 50 MW / 200 MWh battery at Bus 4\.* 

***Table 1\.** Siting: daily savings by location (50 MW / 200 MWh)*

| Bus | Savings (\$/day) | Share of baseline cost | No-battery LMP range (\$/MWh) |
| ----- | ----- | ----- | ----- |
| 4 | 1,071.79 | 0.30% | 15.00 to 40.00 (25.00) |
| 3 | 875.21 | 0.24% | 15.00 to 30.04 (15.04) |
| 2 | 803.20 | 0.22% | 15.00 to 26.42 (11.42) |
| 5 | 348.22 | 0.10% | 10.00 to 15.00 (5.00) |
| 1 | 38.24 | 0.01% | 15.00 to 16.99 (1.99) |

The no-battery cost over the 24-hour profile is \$357,972.81.

***Table 2\.** Sizing at Bus 4 (4-hour battery)*

| Energy (MWh) | Power (MW) | Savings (\$/day) | Share of maximum |
| ----- | ----- | ----- | ----- |
| 100 | 25 | 828 | 63% |
| 200 | 50 | 1,072 | 82% |
| 250 | 62.5 | 1,193 | 91% |
| 300 | 75 | 1,314 | 100% |

## **Key findings**

1. **Location matters more than size for a small battery.** At 50 MW / 200 MWh, the best bus (Bus 4\) saves about 28 times as much as the worst (Bus 1). Battery value depends on where the network is congested, not only on price spreads.  
2. **The ranking follows the daily swing in nodal price.** A battery earns by charging when the price at its bus is low and discharging when it is high. Bus 4 is on the load side of the congested Branch 6 and swings the most (\$15 to \$40/MWh), so it earns the most.  
3. **Energy capacity sets the savings, up to a plateau.** At Bus 4, savings stop growing at about 300 MWh (\$1,314/day, 0.37% of operating cost). A 200 MWh battery captures 82% of that and a 250 MWh battery 91%. Raising power above 50 MW at 200 MWh adds nothing, while a lower rating (25 MW / 8 h) saves 3.4% less because it cannot charge fast enough in the cheap hours.  
4. **The battery does not relieve Branch 6\.** The line is at its limit in all 24 hours with the battery, against 21 without it. Its shadow price is essentially unchanged at the peak (about \$62/MWh) and rises from zero to about \$45/MWh in hours 2 to 4, because overnight charging at Bus 4 uses up the line's slack. This also explains why cheap Gen 5 produces 229.6 MWh less: with Branch 6 full, it is stranded behind the line.  
5. **Duals explain the schedule.** The dual of the state-of-charge equation is the value of stored energy. The battery charges when the LMP equals 0.95 times that value and discharges when the LMP equals the value divided by 0.95. At Bus 4 this gives a flat overnight price of \$31.46 and an evening price of \$39.94.

## **Approach**

| Item | Choice |
| ----- | ----- |
| Problem type | 24-hour DC-OPF, linear program |
| Tools | Pyomo \+ HiGHS |
| Network | PJM 5-bus, branch limits from the case file |
| Load | Synthetic 24-hour profile, 650 to 1,240 MW |
| Battery | One bus at a time, 95% efficiency each way, state of charge between 10% and 100% of capacity, starting and ending the day at 50% |
| Sweeps | Location (5 buses), energy capacity, power rating, initial state of charge |
| Outputs | Dispatch, daily savings, battery schedule, LMPs, Branch 6 shadow price, stored-energy value |

**Validation.** The no-battery model at hour 9 load reproduces the \$17,479.90/h single-period DC-OPF result, which matches PYPOWER in the parent project. The price rule above (LMP equals efficiency times stored-energy value when charging, and value divided by efficiency when discharging) is checked hour by hour in the notebook. PYPOWER has no storage model, so there is no separate reference notebook here.

## **What is in this repo**

```
battery_siting_sizing/
├── README.md
├── requirements.txt
├── notebooks/
│   └── 01_dcopf_bess_siting_sizing.ipynb   # siting, sizing, battery operation and price analysis (Pyomo + HiGHS)
└── images/
```

The notebook runs top to bottom in a few minutes and needs only HiGHS.

## **Theory**

The notebook covers implementation and results, including a short guide to reading duals in a multi-period model. The derivations live in two notes on my website:

* [ED, DC-OPF, and AC-OPF](https://vesuvius-khatiwada.super.site/ed-dc-opf-and-ac-opf): the three formulations, congestion, and how to read duals as LMPs and line shadow prices.  
* [Optimization Foundations for Energy System Modelling](https://vesuvius-khatiwada.super.site/optimization-foundations-for-energy-system-modelling): convexity, Lagrangians, KKT conditions, and a hand-worked economic dispatch.

## **How to run**

```sh
git clone https://github.com/vjkhtw/pjm5-bess-siting.git
cd pjm5-bess-siting
python -m pip install -r requirements.txt
jupyter notebook notebooks/01_dcopf_bess_siting_sizing.ipynb
```

## **Limitations**

The load profile is synthetic and covers one day, and the model has perfect knowledge of the whole day's demand. Every battery starts and ends the day at 50% state of charge, and the capacity at which savings saturate depends on that assumption (with 10% it falls to about 250 MWh). Battery capital cost, degradation and revenue are not included, so the results are operating-cost savings, not an investment case. The DC model ignores losses, reactive power and voltage limits, and there are no ramp limits, startup costs or N-1 contingencies. Nodal prices and shadow prices are marginal values for small changes. Results depend on this test network and its branch limits.

## **Data**

IEEE PES Power Grid Library, pglib\_opf\_case5\_pjm (v23.07), based on F. Li and R. Bo, "Small Test Systems for Power System Economic Studies," IEEE PES General Meeting, 2010\. Licensed CC BY 4.0.

## **Author**

Vesuvius Khatiwada, [portfolio](https://vesuvius-khatiwada.super.site/)

[image1]: images/bess_savings_by_location.png

[image2]: images/savings_against_energy_capacity.png

[image3]: images/Prices_and_Branch_6_shadow_price.png