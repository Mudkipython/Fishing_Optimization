# A Preference-Based Optimization Model for Fishing Planning in Quebec Zone 8

## 📖 1. Project Background
A fishing trip in Quebec Zone 8 involves many intertwined choices that directly shape an angler's outcome and enjoyment. Within limited time and travel capacity, the angler must decide which district to visit, which subset of sites to include, and how to move between them. 

This project builds a Mixed-Integer Programming (MIP) model that translates site information, regulatory limits (open seasons, allowable methods, bag limits), and exogenous user preferences into one coherent plan. The model maximizes the angler's overall utility while ensuring full compliance and realistic multi-modal travel feasibility.

---

## 🧮 2. Utility Function Formulation

The optimization problem centers on the maximization of Total Utility, which is an aggregate of objective environmental quality, subjective preferences, and catch outcomes.

### 2.1 Base Utility
Base Utility evaluates the inherent quality of a specific site-species combination using normalized attributes [0,1]:
* **Habitat Suitability** ($w_{hab}=0.2$)
* **Potential Fish Size** ($w_{size}=0.2$)
* **Threat Level** ($w_{threat}=0.1$)
* **Conservation Status** ($w_{status}=0.1$)

$$U_{i,s}^{base}=w_{hab}\cdot\hat{habitat}_{i}+w_{size}\cdot\hat{size}_{i,s}+w_{threat}\cdot\hat{threat}_{s}+w_{status}\cdot\hat{status}_{s}$$

### 2.2 Preference-Augmented Utility
User preferences for species ($\alpha_{s}$), bait categories ($\beta(t)$), and rod lengths ($\delta(r(i))$) act as binary inputs that activate additive utility bonuses:

$$U_{i,s,t}=U_{i,s}^{base}+\alpha_{s}+\beta(t)+\delta(r(i))$$

### 2.3 Catch and Retention Utility
Every fish caught contributes a base utility, and retaining the fish yields a marginal bonus:

$$U_{i,s,t}^{catch}=0.8\cdot TC_{i,s,t}+0.2\cdot K_{i,s,t}$$

---

## 🛠️ 3. Decision Variables

To model the trade-offs between spatial selection, fishing intensity, catch retention, and travel modes, we define the following decision variables:

| Variable | Definition and Interpretation | Domain |
| :--- | :--- | :--- |
| $g_{d}$ | District indicator. Equal to 1 if district d is selected. | Binary |
| $y_{i}$ | Site visit indicator. Equal to 1 if site i is included in the itinerary. | Binary |
| $x_{i,s,t}$ | Strategy activation. Targets species s using bait t at site i. | Binary |
| $h_{i,s,t}$ | Time allocation. Continuous hours allocated to strategy (i,s,t). | $\mathbb{R}_{\ge0}$ |
| $b_{i,s,t}$ | Retention intention. Intends to keep fish caught via this strategy. | Binary |
| $TC_{i,s,t}$ | Total catch quantity (targeted + bycatch). | $\mathbb{R}_{\ge0}$ |
| $K_{i,s,t}$ | Kept catch. Integer number of fish retained (harvested). | $\mathbb{Z}_{\ge0}$ |
| $v_{i,j}$ | Routing connection. Travels directly from location i to j. | Binary |
| $D_{i,j}$ | Driving mode. Leg (i,j) performed by car (incurs prep penalty). | Binary |
| $W_{i,j}$ | Walking mode. Leg (i,j) performed by walking (< 3km). | Binary |
| $U_{i}$ | Subtour elimination variable (used in MTZ constraints). | $\mathbb{Z}_{\ge0}$ |

---

## 🎯 4. Objective Function

The optimization goal balances the total accumulated utility from fishing and catching against the opportunity cost of time consumed ($\lambda=2.0$ per hour).

$$\max Z=\sum_{(i,s,t)\in\Omega}(U_{i,s,t}\cdot h_{i,s,t}+U_{i,s,t}^{catch})-\lambda\cdot T_{total}$$

Where $T_{total}$ includes active fishing, driving ($\tau_{i,j}^{D}$), walking ($\tau_{i,j}^{W}$), and a fixed preparation penalty ($P_{prep} = 0.5$ hours) for driving:

$$T_{total}=\sum_{(i,s,t)\in\Omega}h_{i,s,t}+\sum_{i\ne j}[(\tau_{i,j}^{D}+P_{prep})\cdot D_{i,j}+\tau_{i,j}^{W}\cdot W_{i,j}]$$

---

## 🚧 5. Constraints Formulation

The model operates within strict temporal, physical, and regulatory boundaries. Pre-filtered legal strategies are defined within set $\Omega$.

| Constraint Focus | Formula | Explanation |
| :--- | :--- | :--- |
| **District** | $\sum_{d}g_{d}=1$ ; $y_{i}\le g_{d(i)}$ | Choose exactly 1 district; visit only sites in it. |
| **Activation** | $x_{i,s,t}\le y_{i}$ | Can only fish at visited sites. |
| **Time Window** | $0.25\cdot x_{i,s,t}\le h_{i,s,t}\le4.0\cdot x_{i,s,t}$ | Fishing time between 0.25h and 4h if active. |
| **Total Catch** | $TC_{i,s,t}=\lambda_{i,s,t}h_{i,s,t}+\sum_{s^{\prime}\ne s}0.3\cdot\lambda_{i,s,t}h_{i,s^{\prime},t}$| Target catch + 0.3 x Bycatch from other species. |
| **Physical Limit** | $K_{i,s,t}\le TC_{i,s,t}$ | Kept $\le$ Total Caught. |
| **Flow** | $\sum_{j}v_{i,j}=y_{i}$ ; $\sum_{j}v_{j,i}=y_{i}$ | Enter and leave every visited site once. |
| **Mode** | $D_{i,j}+W_{i,j}=v_{i,j}$ | Choose Drive (D) OR Walk (W) for each leg. |
| **Time Limit** | $\sum_{\Omega}h+\sum_{i,j}[(\tau^{D}+0.5)D+\tau^{W}W]\le12$ | Fishing + Travel + 0.5h Prep $\le$ 12h. |
| **MTZ** | $U_{i}-U_{j}+\|I\|v_{i,j}\le\|I\|-1$ | Subtour elimination for valid routing. |
| **Bag Limits** | $\sum_{i,t}K_{i,s,t}\le6$ ; $\sum_{i,s,t}K_{i,s,t}\le6$ | Max 6 fish per species; Max 6 fish total per trip. |

---

## 🌐 6. Advanced Extensions
To test the robustness of the solution against St. Lawrence River's stochastic operational realities, the model was extended with:
1.  **Environmental Volatility (Safety)**: A binary filter $\sigma_{d,t}$ re-routes itineraries if wind forecasts exceed safe limits (18 km/h).
2.  **Economic Constraints (Budget)**: Limits the maximum trip cost based on fuel consumption per km and launch fees.
3.  **Social Friction (Congestion)**: Applies a time-dependent penalty $\rho_{i}(t)$ for peak-hour access to popular sites, prompting the model to suggest walk-in or paid private access.

---

## 👨‍💻 Contributors
* Chenguang Sun
* Ruihe Zhang
* Yuyang Chen
* Zhaihan Gong
* Maddy Agrawal
